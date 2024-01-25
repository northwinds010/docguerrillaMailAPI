
## Introdução
Guerrilla Mail fornece uma API JSON através do HTTP. A API é pública e aberta para todos, não necessita de registro ou chaves de API para operar. A API é usada pelo próprio GuerrillaMail para alimentar o site, então GuerrillaMail.com é na verdade um exemplo de como usar a API.

A URL da API está localizada em: https://api.guerrillamail.com/ajax.php

Cada solicitação para a URL acima deve ter um parâmetro 'f', indicando o nome da função. Pode opcionalmente passar 'ip' e 'agent' para indicar o endereço IP e o user-agent do cliente. Seguido pelo(s) argumento(s) conforme exigido pela especificação da função. O resultado será retornado em uma string codificada em JSON.

#### Exemplo: 
```
https://api.guerrillamail.com/ajax.php?f=get_email_address&ip=127.0.0.1&agent=Mozilla_foo_bar
```

Como requisito, o cliente que interage com a API deve ser capaz de salvar, enviar e receber os seguintes parâmetros

`sid_token` - Um ID de sessão. Definido pelo servidor para manter o estado. O cliente deve sempre procurar este parâmetro em cada retorno de função, lembrar dele (se alterado) e, em seguida, fornecê-lo como um parâmetro em cada chamada de API. Este valor pode mudar a cada chamada, e quando muda, indica que uma nova sessão foi iniciada; o cliente precisaria atualizar a interface com novos dados.

`A API pode ter limites de taxa, mas não publicamos nossos limites de taxa. Por favor, use esta API com cuidado.`

## Caminho

1. O usuário acessa a aplicação Guerrilla Mail. O aplicativo é inicializado e chama `get_email_address` para obter o ID da sessão (sid_token), endereço de e-mail e o timestamp do endereço de e-mail. O cliente pode obter um endereço de e-mail aleatório ou especificar a parte local de um endereço que desejam usar.

2. Chamada para `get_email_list` (com offset=0) para obter a página inicial de até 20 e-mails. Observação: essa função deve ser chamada apenas ao abrir a caixa de correio pela primeira vez como parte da inicialização; não deve ser usada para verificar e-mails.

3. É feita uma chamada para `check_email` para verificar novos e-mails. Esta poderia ser uma função que opera em segundo plano.

4. Um novo e-mail chega, o usuário clica nele. O e-mail é recuperado e exibido (é feita uma chamada para `fetch_email`).

5. Para alterar o endereço de e-mail, o usuário pode definir o novo endereço de e-mail (é feita uma chamada para `set_email_user`). Após chamar essa função, o cliente pode obter a lista de e-mails para este novo endereço, verificando o e-mail.

`Observação: quando um usuário muda o endereço de e-mail de um para outro, o endereço de e-mail antigo e seus e-mails NÃO são excluídos. Se o usuário voltar ao endereço antigo, suas mensagens ainda estarão lá até que o endereço expire ou a mensagem tenha 1 hora.`

6. Esquecer o e-mail - informar ao servidor para esquecer o endereço de e-mail atual (mas não excluí-lo) (é feita uma chamada para `forget_me`).

7. Estender o tempo - (descontinuado, todos os endereços de e-mail funcionam o tempo todo) Dizer ao servidor para estender o tempo para o endereço atual (é feita uma chamada para `extend`).

8. E-mail expirado. O cliente deve manter o controle de quando o e-mail expira. Algumas chamadas também retornam um timestamp. O timestamp é um timestamp Unix, indicando quando o endereço foi criado. O cliente pode calcular quantos minutos restam:<br><br>
$
Segundos\ Restantes = 3600 - Timestamp Atual - Timestamp\ do\ E-mail
$

`Uma sessão pode expirar após cerca de 18 minutos de inatividade. Se uma sessão expirar, um novo endereço de e-mail será gerado ao chamar get_email_address, e um usuário sempre pode retornar ao endereço de e-mail antigo ao chamar set_email_user.`

## Autorização

A API do Guerrilla Mail é pública; a autorização não é necessária a menos que você tenha criado um site e adicionado um domínio personalizado a ele. Para criar um site privado, vá para https://grr.la/ryo/guerrillamail.com/ e crie uma conta. Os sites são privados por padrão e você pode adicionar quantos domínios quiser. Se o site estiver configurado como privado, o e-mail para domínios personalizados só será acessível se um token de Autorização válido for fornecido no cabeçalho. (Veja a aba Configuração, configuração "Acesso ao site" para definir seu site como público se você não quiser usar a autorização).

### Chave API para sites privados

Se estiver usando Autorização, sua Chave API será mostrada no painel de controle, sob a aba Configuração, página API - (mantenha em segredo).

### Como autorizar para sites privados

Com cada chamada de API, você precisará passar o seguinte cabeçalho HTTP:

`Authorization: ApiToken 9fc104bfc7f71e7e1aef5479eb87ebe81e651432377a60953d82ff0de7b99c2e`

ApiToken é um valor secreto derivado da chave API, projetado para mudar para cada sessão. Aqui está como ele é derivado:

    1. Chame get_email_address para obter a string sid_token.
    2. Derive um token chamando uma função HMAC, com os seguintes parâmetros: Algoritmo: SHA-256, segredo: chave API, dados: sid_token por exemplo, em PHP, você chamaria: $token = hash_hmac('sha256', $sid_token, $api_key); Finalmente, o token é enviado através da API especificando um cabeçalho HTTP 'Authorization'.

Por exemplo, em PHP

    1. Pegamos um token API, por exemplo, qqXvbMBcZ2e2nniU
    2. Fazemos uma chamada inicial da API get_email_address e obtemos o sid_token de 'b2idl836kj1a106lo34a47el12'
    3. Derivamos o ApiToken assim: $token = hash_hmac('sha256', $sid_token, $api_key); que nos dá: 9fc104bfc7f71e7e1aef5479eb87ebe81e651432377a60953d82ff0de7b99c2e
    
Ao usar a autorização, o parâmetro 'site' também deve ser passado com cada chamada de API.

### Resultado da Autorização
Toda vez que uma solicitação é feita com um cabeçalho de Autorização, o objeto 'auth' será adicionado a qualquer objeto resultante

#### auth:

``` json
{
success:true, 
error_codes:[]
}
```
Success é booleano. Se for verdadeiro, então o array error_codes estará vazio. Os possíveis códigos de erro são:

``` javascript
array(
    'auth-token-missing' => 'Token de autenticação ausente no cabeçalho',
    'auth-token-invalid' => 'Token de autenticação inválido.',
    'auth-unknown-site' => 'Site inválido usado.',
    'auth-session-not-initialized' => 'Sessão não inicializada. Por favor, chame get_email_address primeiro para pegar o token de sessão',
    'auth-api-key-unconfigured' => 'chave api para site não gerada'
)
```


## Funções

Cada solicitação HTTP para https://api.guerrillamail.com/ajax.php é considerada uma chamada de função. O cliente pode usar GET ao consultar dados e POST ao definir ou excluir dados, embora isso não seja estritamente obrigatório. Cada solicitação deve conter o parâmetro 'f' contendo o nome da função.


### get_email_adress

#### Argumentos
`lang`(opcional) - uma string representando o código de idioma. Atualmente suportado: en, fr, nl, ru, tr, uk, ar, ko, jp, zh, zh-hant, de, es, it, pt.

    fr: Francês
    nl: Holandês (Neerlandês)
    ru: Russo
    tr: Turco
    uk: Ucraniano
    ar: Árabe
    ko: Coreano
    jp: Japonês
    zh: Chinês (Simplificado)
    zh-hant: Chinês (Tradicional)
    de: Alemão
    es: Espanhol
    it: Italiano
    pt: Português

`sid_token` (opcional) - O token de ID de sessão usado para manter o estado. A API também verificará um cookie chamado PHPSESSID para este token se não for passado como argumento.

`site` (opcional) - Se você possui seu próprio domínio e deseja acessar seus domínios personalizados, use este identificador de site para o seu site. O padrão é guerrillamail.com (Consulte RYO -> Configuração -> Página API para obter detalhes).

#### Descrição

A função é usada para inicializar uma sessão e definir o cliente com um endereço de e-mail. Se a sessão já existir, ela retornará os detalhes do endereço de e-mail da sessão existente. Se for necessário criar uma nova sessão, ela criará um novo endereço de e-mail aleatoriamente. A função retornará vários argumentos para que o cliente se lembre, incluindo o 'sid_token'. O 'sid_token' é passado para cada chamada subsequente da API para manter o estado.

#### Método alternativo para controlar a sessão:

A sessão também pode ser mantida usando Cookies HTTP. O nome do cookie é ‘PHPSESSID’ e será fornecido no cabeçalho HTTP resultante. O cliente pode armazenar este cookie e enviá-lo sempre que fizer uma chamada à API. Uma nova sessão será criada quando o cookie ‘PHPSESSID’ não for fornecido pelo cliente. A função também gera um novo e-mail de boas-vindas para o usuário.

Para enviar ‘PHPSESSID’ em uma solicitação HTTP, adicione a seguinte linha ao cabeçalho:

plaintext

`Cookie: PHPSESSID=ABC1234\r\n"`

Onde ‘ABC1234’ é o dado para o cookie ‘PHPSESSID’.

O cliente deve sempre observar o cookie ‘PHPSESSID’ para alterações. Se ele mudar, precisa atualizar esse valor e passá-lo em chamadas subsequentes.

#### Retornos

Um objeto com as seguintes propriedades:

`email_addr` - O endereço de e-mail que foi determinado. Se uma sessão anterior forencontrada, será o endereço de e-mail dessa sessão. Um novo endereço de e-mailaleatório será criado.

`email_timestamp` - um carimbo de data/hora UNIX quando o endereço de e-mail foicriado. Usado pelo cliente para controlar a expiração.

`sid_token` - Token de ID de sessão. Você precisaria fornecer esse parâmetro para cadachamada subsequente da API.

**Exemplo**

`https://api.guerrillamail.com/ajax.php?f=get_email_address&lang=pt`

**Retorno**
``` json
{
   "email_addr":"ooswnhld@guerrillamailblock.com",
   "email_timestamp":1706126364,
   "alias":"tktj44+cd3ntf8ss3k3s",
   "sid_token":"hq6fs3e13kfdmg131dfi4qk277"
}
```

### set_email_user

#### Argumentos

`email_user`` - String, máximo de 74 caracteres. A "parte local" de um endereço dee-mail. Exemplo: test@guerrillamailblock.com, a parte 'email_user' seria 'test'

`lang` - String. O código do idioma, por exemplo, 'en' (inglês).

`sid_token` - Passe isso para manter o estado; o valor para este argumento é retornadoapós chamar a função get_email_address.

Nota: Você não precisa passar o cookie PHPSESSID se passar o argumento 'sid_token'.

`site` (opcional) - Se você possui seu próprio domínio e deseja acessar seus domíniospersonalizados, use este identificador de site para o seu site. O padrão éguerrillamail.com (Consulte RYO -> Configuração -> Página API para obter detalhes).

#### Descrição

Define o endereço de e-mail para um endereço de e-mail diferente. Se o e-mail já foi definido, ele receberá um tempo adicional de 60 minutos. Caso contrário, será gerado um novo endereço de e-mail se o endereço de e-mail não estiver no banco de dados.

#### Retornos

Semelhante a get_email_address, esta função retorna um objeto com as seguintes propriedades:

`email_addr` - O endereço de e-mail que foi definido.

`email_timestamp` - um carimbo de data/hora UNIX quando o endereço de e-mail foicriado. Usado pelo cliente para controlar a expiração.

`sid_token` - Token de ID de sessão.

`site` - (string) O identificador do site correspondido.

`site_id` - (int) O ID do site ao qual o domínio pertence.

`alias` - (string) Esta é a versão embaralhada de email_addr, por exemplo, "fcs5d+vgdtknvsyt4". O endereço embaralhado é um alias para email_addr, então todos ose-mails enviados para alias chegarão à caixa de entrada de email_addr. Os endereçosembaralhados são usados para mascarar o email_addr, tornando difícil para outra pessoasaber qual email_addr foi usado.

`alias_error` (string) Mensagem de erro se o alias não puder ser definido.

**Exemplo**

`https://api.guerrillamail.com/ajax.php?f=set_email_user&email_user=test&site=guerrillamail.com&lang=pt&sid_token=lmmb0hfqa6qjoduvr2vdenas62`

**Retorno**
``` json
{
   "alias_error":"",
   "alias":"tkp74n+6u3vjec",
   "email_addr":"test@guerrillamailblock.com",
   "email_timestamp":1706127584,
   "site_id":1,
   "sid_token":"lmmb0hfqa6qjoduvr2vdenas62",
   "site":"emjd",
   "auth":{
      "success":true,
      "error_codes":[
         
      ]
   }
}
```
### check_email

#### Argumentos

`sid_token` - [obrigatório] Token de ID de sessão retornado pela função `get_email_address`.

`seq` - [obrigatório] O número de sequência (id) do e-mail mais antigo.


#### Descrição


Verifica a presença de novos e-mails no servidor. Retorna uma lista das mensagens mais recentes. O tamanho máximo da lista é de 20 itens.

O cliente não deve verificar os e-mails com muita frequência para não sobrecarregar o servidor. Não verifique se o e-mail expirou; a rotina de verificação de e-mails deve ser pausada se o e-mail tiver expirado.

**Retorno**

`list` - Uma lista de mensagens, representada como uma matriz de objetos. Cada objeto possui as seguintes propriedades:

    - mail_id;
    - mail_from (endereço de e-mail do remetente),
    - mail_subject;
    - mail_excerpt; (trecho do e-mail),
    - mail_timestamp; (um carimbo de data/hora UNIX),
    - mail_read; (1 se lido, 0 se não lido),
    - mail_date;
`Observação: ‘mail_subject’ e ‘mail_excerpt’ são escapados usando Entidades HTML.`

`count` - O número de e-mails correspondidos. Isso pode ser muito mais do que o máximo que pode ser retornado (20)! Este número indica o total de novos e-mails no banco de dados. (Se você quiser obter os e-mails após os primeiros 20, faça outra chamada com a função `get_older_list`)

`email` - O endereço de e-mail para o qual a lista é destinada. Ex: test@guerrillamailblock.com. O cliente precisaria observar essa variável para detectar quaisquer alterações e sincronizar o endereço.

`ts` - O carimbo de data/hora do endereço de e-mail, momento em que o endereço de e-mail foi criado. O cliente pode usar essa variável para sincronizar o tempo. Os e-mails expiram após 60 minutos.

`sid_token` - Token de ID de sessão

`users` - o número de usuários visualizando esta caixa de entrada (não exato)

**Exemplo**

`https://api.guerrillamail.com/ajax.php?f=check_email&seq=0&sid_token=lmmb0hfqa6qjoduvr2vdenas62`

**Retorno**

``` json

{
   "list":[
      {
         "mail_from":"no-reply@guerrillamail.com",
         "mail_timestamp":0,
         "mail_read":0,
         "mail_date":"22:23:29",
         "reply_to":"",
         "mail_subject":"Bem-Vindo a Guerrilla Mail",
         "mail_excerpt":"Prezado Us\u00faario,\r\n\r\nObrigado por usar Guerrilla Mail - seu endere\u00e7o de e-mail tempor\u00e1rio amigo e aliado de luta contra o spam!\r\n\r\nSeu endere\u00e7o de e-mail descart\u00e1vel foi criado e est\u00e1 pronto para",
         "mail_id":1,
         "att":0,
         "content_type":"text",
         "mail_recipient":"ueokpehl",
         "source_id":0,
         "source_mail_id":0,
         "mail_body":"Prezado Us\u00faario,\r\n\r\nObrigado por usar Guerrilla Mail - seu endere\u00e7o de e-mail tempor\u00e1rio amigo e aliado de luta contra o spam!\r\n\r\nSeu endere\u00e7o de e-mail descart\u00e1vel foi criado e est\u00e1 pronto para usar.\r\n\r\nE-mail: ueokpehl@sharklasers.com\r\n\r\nDicas & Notas:\r\n\r\n- Voc\u00ea poder\u00e1 mudar este endere\u00e7o de e-mail! Coloque o mouse sobre a ID da Caixa de Entrada no topo dessa p\u00e1gina e clique para editar.\r\n\r\n- Quando voc\u00ea mudar para um novo endere\u00e7o, o endere\u00e7o antigo ir\u00e1 permanecer dispon\u00edvel, basta reverter e mudar para ele de novo. (Nota: o e-mail \u00e9 mantido durante 1 hora)\r\n\r\n- Esperando pelo seu correio? N\u00e3o precisa refrescar a p\u00e1gina, os novos e-mails ser\u00e3o adicionados \u00e0 lista assim que entrarem.\r\n\r\n- Todos os e-mails ser\u00e3o eliminados ap\u00f3s 1 hora.\r\n\r\nClique em \"\u00ab Voltar para a caixa de entrada\" para regressar \u00e0 lista do correio\r\n\r\nObrigado,\r\n\r\nEquipe da Guerrilla Mail\r\nhttp:\/\/www.guerrillamail.com\/\r\n\r\n\"Pode baixar gr\u00e1tis, mas voc\u00ea precisa de fornecer o seu endere\u00e7o de e-mail para que possamos, inevitavelmente, tentar lhe vender coisas no futuro? guerrillamail.com FTW!\"\r\n\r\nConecte-se conosco:\r\nhttps:\/\/www.facebook.com\/GuerrillaMail\r\nhttps:\/\/twitter.com\/GuerrillaMail",
         "size":1199
      }
   ],
   "count":"0",
   "email":"ueokpehl@guerrillamailblock.com",
   "alias":"tktpmg+3g52aa3ff9178",
   "ts":1706135009,
   "sid_token":"lmmb0hfqa6qjoduvr2vdenas62",
   "stats":{
      "sequence_mail":"73,145,039",
      "created_addresses":38163242,
      "received_emails":"16,017,193,670",
      "total":"15,944,048,631",
      "total_per_hour":"34800"
   },
   "auth":{
      "success":true,
      "error_codes":[
         
      ]
   }
}
```

### get_email_list

#### Argumentos

`sid_token` - [obrigatório] Token de ID de sessão retornado pela função get_email_address

`offset` - [obrigatório] Quantos e-mails começar a partir (ignorar). Começa a partir de 0.

`seq` - [opcional] O número da sequência (id) do primeiro e-mail.

#### Descrição

Obtém no máximo 20 mensagens a partir do deslocamento especificado. O deslocamento de 0 irá buscar uma lista dos primeiros 10 e-mails, o deslocamento de 10 buscará uma lista dos próximos 10 e assim por diante.

Esta função é útil para popular a lista inicial de e-mails. Se desejar verificar novos e-mails, use a função `check_mail`. Observação: Quando retornados, o assunto e o trecho do e-mail são escapados usando Entidades HTML.

**Retorno**

`list` - Uma lista de mensagens, representada como uma matriz de objetos. Cada objeto possui as seguintes propriedades:

    - mail_id;
    - mail_from (endereço de e-mail do remetente),
    - mail_subject;
    - mail_excerpt; (trecho do e-mail),
    - mail_timestamp; (um carimbo de data/hora UNIX),
    - mail_read; (1 se lido, 0 se não lido),
    - mail_date;
`Observação: ‘mail_subject’ e ‘mail_excerpt’ são escapados usando Entidades HTML.`

`count` - O número de e-mails correspondidos. Isso pode ser muito mais do que o máximo que pode ser retornado (20)! Este número indica o total de novos e-mails no banco de dados. (Se você quiser obter os e-mails após os primeiros 20, faça outra chamada com a função `get_older_list`)

`email` - O endereço de e-mail para o qual a lista é destinada. Ex: test@guerrillamailblock.com. O cliente precisaria observar essa variável para detectar quaisquer alterações e sincronizar o endereço.

`ts` - O carimbo de data/hora do endereço de e-mail, momento em que o endereço de e-mail foi criado. O cliente pode usar essa variável para sincronizar o tempo. Os e-mails expiram após 60 minutos.

`sid_token` - Token de ID de sessão

`users` - o número de usuários visualizando esta caixa de entrada (não exato)

**Exemplo**

`https://api.guerrillamail.com/ajax.php?f=get_email_list&offset=0&sid_token=lmmb0hfqa6qjoduvr2vdenas62`

**Retorno**
``` json
{
   "list":[
      {
         "mail_from":"no-reply@guerrillamail.com",
         "mail_timestamp":0,
         "mail_read":0,
         "mail_date":"22:23:29",
         "reply_to":"",
         "mail_subject":"Bem-Vindo a Guerrilla Mail",
         "mail_excerpt":"Prezado Us\u00faario,\r\n\r\nObrigado por usar Guerrilla Mail - seu endere\u00e7o de e-mail tempor\u00e1rio amigo e aliado de luta contra o spam!\r\n\r\nSeu endere\u00e7o de e-mail descart\u00e1vel foi criado e est\u00e1 pronto para",
         "mail_id":1,
         "att":0,
         "content_type":"text",
         "mail_recipient":"ueokpehl",
         "source_id":0,
         "source_mail_id":0,
         "mail_body":"Prezado Us\u00faario,\r\n\r\nObrigado por usar Guerrilla Mail - seu endere\u00e7o de e-mail tempor\u00e1rio amigo e aliado de luta contra o spam!\r\n\r\nSeu endere\u00e7o de e-mail descart\u00e1vel foi criado e est\u00e1 pronto para usar.\r\n\r\nE-mail: ueokpehl@sharklasers.com\r\n\r\nDicas & Notas:\r\n\r\n- Voc\u00ea poder\u00e1 mudar este endere\u00e7o de e-mail! Coloque o mouse sobre a ID da Caixa de Entrada no topo dessa p\u00e1gina e clique para editar.\r\n\r\n- Quando voc\u00ea mudar para um novo endere\u00e7o, o endere\u00e7o antigo ir\u00e1 permanecer dispon\u00edvel, basta reverter e mudar para ele de novo. (Nota: o e-mail \u00e9 mantido durante 1 hora)\r\n\r\n- Esperando pelo seu correio? N\u00e3o precisa refrescar a p\u00e1gina, os novos e-mails ser\u00e3o adicionados \u00e0 lista assim que entrarem.\r\n\r\n- Todos os e-mails ser\u00e3o eliminados ap\u00f3s 1 hora.\r\n\r\nClique em \"\u00ab Voltar para a caixa de entrada\" para regressar \u00e0 lista do correio\r\n\r\nObrigado,\r\n\r\nEquipe da Guerrilla Mail\r\nhttp:\/\/www.guerrillamail.com\/\r\n\r\n\"Pode baixar gr\u00e1tis, mas voc\u00ea precisa de fornecer o seu endere\u00e7o de e-mail para que possamos, inevitavelmente, tentar lhe vender coisas no futuro? guerrillamail.com FTW!\"\r\n\r\nConecte-se conosco:\r\nhttps:\/\/www.facebook.com\/GuerrillaMail\r\nhttps:\/\/twitter.com\/GuerrillaMail",
         "size":1199
      }
   ],
   "count":"0",
   "email":"ueokpehl@guerrillamailblock.com",
   "alias":"tktpmg+3g52aa3ff9178",
   "ts":1706135009,
   "sid_token":"lmmb0hfqa6qjoduvr2vdenas62",
   "stats":{
      "sequence_mail":"73,145,254",
      "created_addresses":38163242,
      "received_emails":"16,017,216,866",
      "total":"15,944,071,612",
      "total_per_hour":"35500"
   },
   "auth":{
      "success":true,
      "error_codes":[
         
      ]
   }
}
```


### fetch_email

#### Argumentos

`sid_token` - Token de ID de sessão retornado pela função get_email_address.

`email_id` - O ID do e-mail a ser recuperado.

#### Descrição

Obtém o conteúdo de um e-mail

Retorna

Retorna false se não encontrado, ou uma matriz com as seguintes propriedades:

`mail_id` - um inteiro.

`mail_from` - endereço de e-mail do remetente.

`mail_recipient` - endereço de e-mail do destinatário.

`mail_subject` - string UTF-8 do assunto do e-mail.

`mail_excerpt` - um pequeno trecho do e-mail.

`mail_body` - A parte da mensagem do e-mail. Pode conter HTML filtrado conformdescrito acima.

`mail_timestamp` - carimbo de data/hora Unix da chegada.

`mail_date` - Data de chegada no seguinte formato se for inferior a 24 horasH:M:S, ou Y-M-D se for mais antigo que 24 horas.

`mail_read` - 0 ou 1 indicando se o e-mail foi recuperado anteriormente.

`content_type` - Tipo MIME para o mail_body, pode ser text/plain ou text/html.

`sid_token` - Token de ID de sessão

**Exemplo**

`https://api.guerrillamail.com/ajax.php?f=fetch_email&email_id=1&sid_token=lmmb0hfqa6qjoduvr2vdenas62`

**Retorno**

```json
{
   "mail_from":"no-reply@guerrillamail.com",
   "mail_timestamp":0,
   "mail_read":0,
   "mail_date":"23:32:14",
   "reply_to":"",
   "mail_subject":"Bem-Vindo a Guerrilla Mail",
   "mail_excerpt":"Prezado Us&uacute;ario,\r\n\r\nObrigado por usar Guerrilla Mail - seu endere&ccedil;o de e-mail tempor&aacute;rio amigo e aliado de luta contra o spam!\r\n\r\nSeu endere&ccedil;o de e-mail descart&aacute;vel foi criado e est&aacute; pronto para",
   "mail_id":1,
   "att":0,
   "content_type":"text",
   "mail_recipient":"dypxplrf",
   "source_id":0,
   "source_mail_id":0,
   "mail_body":"<pre>Prezado Us&uacute;ario,\r\n\r\nObrigado por usar Guerrilla Mail - seu endere&ccedil;o de e-mail tempor&aacute;rio\namigo e aliado de luta contra o spam!\r\n\r\nSeu endere&ccedil;o de e-mail descart&aacute;vel foi criado e est&aacute; pronto para usar.\r\n\r\nE-mail: dypxplrf@sharklasers.com\r\n\r\nDicas &amp; Notas:\r\n\r\n- Voc&ecirc; poder&aacute; mudar este endere&ccedil;o de e-mail! Coloque o mouse sobre a ID\nda Caixa de Entrada no topo dessa p&aacute;gina e clique para editar.\r\n\r\n- Quando voc&ecirc; mudar para um novo endere&ccedil;o, o endere&ccedil;o antigo ir&aacute;\npermanecer dispon&iacute;vel, basta reverter e mudar para ele de novo. (Nota: o\ne-mail &eacute; mantido durante 1 hora)\r\n\r\n- Esperando pelo seu correio? N&atilde;o precisa refrescar a p&aacute;gina, os novos\ne-mails ser&atilde;o adicionados &agrave; lista assim que entrarem.\r\n\r\n- Todos os e-mails ser&atilde;o eliminados ap&oacute;s 1 hora.\r\n\r\nClique em \"&laquo; Voltar para a caixa de entrada\" para regressar &agrave; lista do\ncorreio\r\n\r\nObrigado,\r\n\r\nEquipe da Guerrilla Mail\r\n<a href=\"http:\/\/www.guerrillamail.com\/\">http:\/\/www.guerrillamail.com\/<\/a>\r\n\r\n\"Pode baixar gr&aacute;tis, mas voc&ecirc; precisa de fornecer o seu endere&ccedil;o de\ne-mail para que possamos, inevitavelmente, tentar lhe vender coisas no\nfuturo? guerrillamail.com FTW!\"\r\n\r\nConecte-se conosco:\r\n<a href=\"https:\/\/www.facebook.com\/GuerrillaMail\">https:\/\/www.facebook.com\/GuerrillaMail<\/a>\r\n<a href=\"https:\/\/twitter.com\/GuerrillaMail\">https:\/\/twitter.com\/GuerrillaMail<\/a><\/pre>",
   "size":1199,
   "ref_mid":"1:493d10a35a40d805be469e9c12242cae92361741",
   "sid_token":"lmmb0hfqa6qjoduvr2vdenas62",
   "auth":{
      "success":true,
      "error_codes":[
         
      ]
   }
}

```

### forget_me

#### Argumentos

`sid_token` - [obrigatório] Token de ID de sessão retornado pela função get_email_address
`email_addr` - o endereço de e-mail a ser esquecido. O email_addr deve ter o mesmo valor retornado por set_email_user ou get_email_address

Descrição

Esquece o endereço de e-mail atual. Isso não interromperá a sessão; a sessão existente será mantida. Uma chamada subsequente para get_email_address buscará um novo endereço de e-mail, ou o cliente pode chamar set_email_user para definir um novo endereço. Tipicamente, um usuário gostaria de definir um novo endereço manualmente após clicar no botão 'esquecer-me'.

**Exemplo**

`https://api.guerrillamail.com/ajax.php?f=forget_me&sid_token=lmmb0hfqa6qjoduvr2vdenas62`

**Retorno**

``` json
true
```

### del_email

#### Argumentos

`sid_token` - Token de ID de sessão retornado pela função get_email_address

`email_ids` - array ou um número inteiro, no seguinte formato: email_ids[]=425&
email_ids[]=426&email_ids[]=427

Onde 425, 426 e 427 são os IDs dos e-mails a serem excluídos. Em uma solicitação HTTP, esse parâmetro seria codificado da seguinte forma:

`email_ids%5B%5D=425&email_ids%5B%5D=426&email_ids%5B%5D=427`


Descrição

Exclui os e-mails do servidor.

#### Retorno

`deleted_ids` - um array de IDs de e-mails excluídos.

**Exemplo**

`https://api.guerrillamail.com/ajax.php?f=del_email&sid_token=lmmb0hfqa6qjoduvr2vdenas62&email_ids%5B%5D=134662652`

**Retorno**

```json
{
   "deleted_ids":[
      "134662652"
   ],
   "auth":{
      "success":true,
      "error_codes":[
         
      ]
   }
}
```

### get_older_list

#### Argumentos

`sid_token` - Token de ID de sessão retornado pela função get_email_address.

`seq` - Número inteiro. Obtém e-mails que têm um ID inferior ao ‘seq’.

`limit` - Número inteiro indicando quantos e-mails buscar no máximo.

#### Descrição

A ideia desta função é obter alguns e-mails mais antigos (com ID inferior ao ‘seq’). Como isso pode ser útil? Digamos que tenhamos mais de 1 página de e-mails. Em seguida, excluímos alguns e-mails da primeira página, deixando alguns espaços vazios na parte inferior. Em vez de chamar fetch_email_list, chamamos isso para ser mais eficiente e obtermos exatamente o número de e-mails que precisamos para preencher esses espaços.
Retorna

A mesma estrutura retornada por fetch_email_list.
