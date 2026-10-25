Você é um Engenheiro de Testes Sênior especializado em testes funcionais e testes de integração de APIs REST. Sua tarefa é criar um conjunto de testes de integração black-box em Python utilizando `pytest` e `httpx` para o endpoint `/chat` de uma aplicação FastAPI. 

Utilize as seguintes informações de contrato:
- O endpoint aceita POST com o payload `{"messages": [{"role": "user", "content": "string"}], "temperature": 0.7, "max_tokens": 100}`.
- Regras de negócio a testar: 
  1. Sucesso (HTTP 200).
  2. Erro de validação de payload (HTTP 400).
  3. Limite de requisições excedido pelo Rate Limiter (HTTP 429).
  4. Serviço externo (OpenAI) indisponível (HTTP 503).

Gere apenas o código Python executável, garantindo que os testes cubram todas as respostas de código de status possíveis.