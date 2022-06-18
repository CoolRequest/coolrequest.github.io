TODO:
Pensar em como estruturar o código do webhook. 
Tem que declarar um módulo ou classe que receba um array de updates e faça o que tem que fazer.
Em outro lugar (pode ser o próprio arquivo da view), fazer o parse do conteúdo recebido no webhook e chamar esse método.
Depois, continuar seguindo o fluxo pelo Django Tutorial até chegar em alguma coisa testável
Quando chegar, dar um deploy no heroku da cool request





Primeiro: explicar por que precisa de um bot

Depois: como criar o bot
https://core.telegram.org/bots#3-how-do-i-create-a-bot
- iniciar conversa com @botfather
- nome: CR Test Bot 01
- username: crdev_test1_bot
  Use this token to access the HTTP API:
  bla
  Keep your token secure and store it safely, it can be used by anyone to control your bot.

---

Dúvidas
- como tornar o conteúdo "Privado"? (não quero que outras pessoas possam incluir esse bot em seus grupos)

----

Ler mensagens na fila:
https://api.telegram.org/botTOKEN/getUpdates
curl "https://api.telegram.org/botTOKEN/getUpdates?offset=543829395" # offset = update_id of the first update to be returned

EnviarMensagem:
curl "https://api.telegram.org/botTOKEN/sendMessage?chat_id=5366032645&text=hello"

---

Procedimento do ponto de vista do usuário
1. Incluir o bot no grupo
2. Mandar uma mensagem no grupo: "/notify"
3. Abrir a aplicação web
4. Verificar a autorização pendente na lista
5. Confirmar

Do ponto de vista da aplicação (webhook)
- Quando recebe uma mensagem com "/notifications":
  - salva o chat_id, data, hora, e sender em uma lista
  - envia um reply dizendo que está pendente de autorização
  - em desenvolvimento: ou usa um job pra ler as mensagens ou chama manualmente um método

Do ponto de vista da aplicação (web)
- exibe lista de autorizações pendentes
- quando o usuário clica em autorizar:
  - salva aquele chat_id na lista de destinatarios
  - manda mensagem para aquele chat_id dizendo que as notificações estão habiltadas para esta conversa

Do ponto de vista da aplicação (notificacao)
- envia mensagens para os chat_ids da lista
