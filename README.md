<?php

	if ( ! defined ('ABSPATH') ) exit("Error 403");

  class ApiController extends Controller{

    public function consulta2(){

    $Benchmark = new Benchmark();
    $Benchmark->start();

    if (!isset($_GET['key'])) {
      echo json_encode(Mensagem::api('chave'));
      die();
    }

    if (!isset($_GET['nb'])) {
      echo json_encode(Mensagem::api('digite-nb'));
      die();
    }

    if (!isset($_GET['key']) && !isset($_GET['nb'])) {
        echo json_encode(Mensagem::api('preencha-campos2'));
      die();	
    }

    $uid_usuario = $_GET['key'];
    $nb = preg_replace('/[^0-9]/', '', $_GET['nb']);

    Log::save_log("consulta2", 'INICIO', $nb, $uid_usuario, 'pre_token', '');

    $apiModel = $this->load_model('api_model');
    $infoApi = $apiModel->api_informacoes_usuario($uid_usuario);
    $acessoUsuario = array_column($infoApi['acesso'], 'acesso');
    $modulo = array_column($infoApi['modulo'], 'modulo');

    if (!$infoApi['usuario']) {
      echo json_encode(Mensagem::api('chave-inva'));
      die();
    }

    $modelSql = $this->load_model('sqlsrv_model');
    $especieConsignavel = $modelSql->consultaNb($nb);

    if (!in_array('sup', $acessoUsuario) || !in_array('consultaplus', $modulo)) {
      Validacao::horarioDeConsultaExtratoApi();
    }

    Validacao::acessoApi($infoApi, 'consulta2');

    if (!in_array('sup', $acessoUsuario)) {
      Validacao::validarEspecieConsignavel($especieConsignavel, 'api');
    }

    Validacao::validarNb($nb, 'api');

    $empresaModel = $this->load_model('empresa_model');
    $usuarioModel = $this->load_model('usuario_model');
    $loteModel = $this->load_model('lote_model');  
    $modelSql = $this->load_model('sqlsrv_model');

    $creditos = $empresaModel->credito_geral($uid_usuario);
    $hitAtual = $usuarioModel->hit_atual($uid_usuario, 'consulta2');
    $prioridade_lote = $empresaModel->informacoes_gerais($infoApi['usuario']['uid_empresa']);
    $hitAtualEmpresa = $empresaModel->hit_atual_empresa($infoApi['usuario']['uid_empresa'], 'consulta2');
    $prioridade_lote = $empresaModel->informacoes_gerais($infoApi['usuario']['uid_empresa']);

    $hitAtual['hit'] ?? $hitAtual['hit'] = 0;
    $hitAtualEmpresa['hit'] ?? $hitAtualEmpresa['hit'] = 0;

    if (Credito::apiHit($creditos['credito'], $creditos['credito_empresa'], $hitAtual['hit'], $hitAtualEmpresa['hit'], 'consulta2')) {

      if (Credito::apiCredito($creditos['credito'], $creditos['credito_empresa'], 'consulta2')) {

        if (STATUS_CONSULTACACHE_ONLINE) {

          $cache = $loteModel->consultaNbOfflineLogCache($nb);

          if (!empty($cache)) {

            $usuarioModel->hit_consulta($uid_usuario, 'consulta2');
            $empresaModel->hit_consulta_empresa($infoApi['usuario']['uid_empresa'], 'consulta2');
            $pesquisa = json_decode($cache['json'], true);
            Validacao::validaIdade($pesquisa, $acessoUsuario, $modulo, 'api');
            $ddb = $modelSql->consultaNb_ddb_mae($nb);
            $pesquisa = ConvertJson::tratamentoJsonOnline($pesquisa, $ddb, 'api');
            $pesquisa = ConvertJson::unsetKey($pesquisa, 'api(Cache)');

            $retorno = $Benchmark->finalized();
            $pesquisa = ConvertJson::resultInfo($pesquisa, $uid_usuario, $creditos['credito']['consulta2'], $retorno['time']);
            Log::save_log("consulta2(Cache)", 'SUCESSO', $uid_usuario.' consultou o beneficio: '.$nb, $uid_usuario, 'api', '');
            echo json_encode($pesquisa);
            die();
         }
      }

      if (STATUS_CONSULTA2) {

        $pesquisa = Curl::consulta_online_fila($infoApi['key']['grupo'], $nb, $uid_usuario, $infoApi['key']['maximo'], $infoApi['usuario']['uid_empresa'], 0);

        switch (@$pesquisa['code']) {
          case '403':

            Log::save_log("consulta2", "ERRO", $nb.' = 403', $uid_usuario, 'api', '');
            $loteModel->controle_erro_nb($nb, '403');

            $uid_usuario = array('0662a3cbf1', '60631f749fff9', '60632ec66c452', '60aba47262b73');
            $empresa = $empresaModel->informacoes_gerais('6058e2c761eba');
            $pesquisa = Curl::saldo_consulta2();

              if ($pesquisa <= 30000) {
                foreach ($uid_usuario as $uid) {
                  $token_notificacao = bin2hex(random_bytes(5));
                  $socketio = $socketModel->select_uid_socket($uid);
                  $info = Curl::notificacao_socketio(1, 'consulta2/saldo', 'Seu saldo de Consulta2 está acabando: '.$pesquisa, $socketio['uid_socket'], $socketio['token_notificacao']);
                  if ($info['error'] == 1) {
                      $notificacaoModel->cadastro_notificacao($uid, 'Seu saldo de Consulta2 está acabando: '.$pesquisa, $token_notificacao, 'lote', 'lote/lote/', null, null, null);
                  }
                }
                Email::saldoConsulta2($pesquisa, 'https://inss.api.dataconsig.com.br/v1/billing/balance?apikey=eac9819e5b513e3ec70453078910190e', $empresa['empresa']['email_servidor'], $empresa['empresa']['senha'], $empresa['empresa']['servidor_smtp'], $empresa['empresa']['porta_smtp']);
                echo json_encode(Mensagem::api('503'));
                die();
              }
            break;
            case '404':
              Log::save_log("consulta2", 'ERRO', $nb.' = 404', $uid_usuario, 'api', '');
              $loteModel->controle_erro_nb($nb, '404');

              echo json_encode(Mensagem::api('404'));
              die();
            break;
            case '400':
              Log::save_log("consulta2", 'ERRO', $nb.' = 400', $uid_usuario, 'api', '');
              $loteModel->controle_erro_nb($nb, '400');
              echo json_encode(Mensagem::api('429'));
              die();
            break;
            case '401':
              Log::save_log("consulta2", 'ERRO', $nb.' = 401', $uid_usuario, 'api', '');
              $loteModel->controle_erro_nb($nb, '401');
              echo json_encode(Mensagem::api('400'));
              die();
            break;
            case '429':
              Log::save_log("consulta2", 'ERRO', $nb.' = 429', $uid_usuario, 'api', '');
              echo json_encode(Mensagem::api('429'));
              die();
            break;
            case '503': 
                Log::save_log("consulta2", 'ERRO', $nb.' = 503', $uid_usuario, 'api', '');
                $loteModel->controle_erro_nb($nb, '503');
                echo json_encode(Mensagem::api('503'));
                die();
            break;
            default:
              if (isset($pesquisa['nome'])) {
                  try {
                    $ddb = $modelSql->consultaNb_ddb_mae($nb);
                    $pesquisa = ConvertJson::tratamentoJsonOnline($pesquisa, $ddb, 'api', '');

                      $loteModel->saveConsultaNovo($nb, $pesquisa);
                      $loteModel->saveConsultaNovo_DBOFF($nb, $pesquisa, 3);
                      Validacao::validaIdade($pesquisa, $acessoUsuario, $modulo, 'api', '');
                      $usuarioModel->hit_consulta($uid_usuario, 'consulta2');
                      $empresaModel->hit_consulta_empresa($infoApi['usuario']['uid_empresa'], 'consulta2');                  

                      $Benchmark->finalized();
                      $retorno = $Benchmark->benchmark_show();
                      $pesquisa = ConvertJson::resultInfo($pesquisa, $uid_usuario, $creditos['credito']['consulta2'], $retorno['time']);
                      $pesquisa = ConvertJson::unsetKey($pesquisa, 'api(Online)');
                      Log::save_log("consulta2", "SUCESSO", $uid_usuario.' consultou o beneficio: '.$nb, $uid_usuario, 'api', '');

                      echo json_encode($pesquisa);
                      die();
                  } catch (Exception $e) {
                      Log::save_log("consulta2", 'ERRO', $nb.' - '.$e, $uid_usuario, 'api', '');
                      echo json_encode(Mensagem::catch_api($e));
                      die();
                  }
            }else{
              Log::save_log("consulta2", 'ERRO', $nb.' - 404', $uid_usuario, 'api', '');
              echo json_encode(Mensagem::api('NB-429'));
              die();
            }
                break;                              
            }
          }else{
            echo json_encode(Mensagem::api('manutencao-consulta'));
              die();
          }	
        }
      }
    }
	}
?>
