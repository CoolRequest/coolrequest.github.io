TODO:
- escrever a o passo 4 - registration logic (texto + app)
- criar projeto da aplicação demo no github
- fazer deploy no heroku da cool request


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

Ler mensagens na fila:
https://api.telegram.org/botTOKEN/getUpdates
curl "https://api.telegram.org/botTOKEN/getUpdates?offset=543829395" # offset = update_id of the first update to be returned

EnviarMensagem:
curl "https://api.telegram.org/botTOKEN/sendMessage?chat_id=5366032645&text=hello"

Habilitar webhook:

curl "https://api.telegram.org/botTOKEN/setWebhook?url=https://blablabla.com/telegram/webhook"

 # just for testing
    post 'telegram/webhook'

---