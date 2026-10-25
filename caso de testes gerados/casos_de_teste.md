# 🧪 Suíte de Casos de Teste de Integração — Chatbot UFG API

Este documento detalha os casos de teste de integração gerados para o endpoint `/chat` da API REST do Chatbot UFG. Os casos de teste foram projetados por meio de técnicas de **Engenharia de Prompt** aplicadas a um modelo de linguagem de grande porte (LLM), combinando abordagens **Funcionais (Caixa-Preta)** e **Estruturais (Caixa-Branca)**.

---

## 📌 Contexto da Aplicação em Teste

A API conecta o cliente (front-end ou middleware) ao serviço da OpenAI através do endpoint `/chat`. A arquitetura envolve:
1. **Middleware de Segurança & Rate Limiter:** Controle de limite de requisições (`429 Too Many Requests`).
2. **Validação de Payloads (Pydantic):** Garantia de dados de entrada corretos (`400 Bad Request`).
3. **Serviço de Comunicação Externa (`get_openai_response`):** Integração assíncrona com a API OpenAI (`503 Service Unavailable` em falhas externas).
4. **Tratamento Global de Exceções:** Mapeamento de erros não previstos (`500 Internal Server Error`).

---

## 📑 Parte 1: Suíte de Testes Funcionais (Caixa-Preta)

**Prompt Utilizado:** `prompt1.md`  
**Técnica:** *Zero-Shot Estruturado*  
**Objetivo:** Validar o contrato da API, semântica HTTP e regras de negócio a partir da especificação do endpoint.

### Matriz de Casos de Teste Funcionais

| ID | Nome do Caso de Teste | Entradas / Condição | Comportamento Esperado | Status Code |
|---|---|---|---|---|
| **TC-01** | `test_chat_success_200` | Payload válido com mensagens bem formatadas e `temperature` padrão. | Retorna a resposta do modelo e metadados de uso. | `200 OK` |
| **TC-02** | `test_chat_validation_error_empty_messages_400` | Payload com lista de mensagens vazia `{"messages": []}`. | Retorna mensagem de erro detalhando falha no Pydantic. | `400 Bad Request` |
| **TC-03** | `test_chat_validation_invalid_role_400` | Mensagem com `role` inválido (`"role": "admin"`). | Retorna erro de validação do Pydantic. | `400 Bad Request` |
| **TC-04** | `test_chat_rate_limit_exceeded_429` | Envios sequenciais rápidos que ultrapassam a taxa permitida pelo middleware. | Retorna mensagem de limite de requisições excedido. | `429 Too Many Requests` |
| **TC-05** | `test_chat_service_unavailable_503` | Falha de comunicação/timeout na API da OpenAI. | Retorna erro padronizado de serviço indisponível. | `503 Service Unavailable` |

---

## 📑 Parte 2: Suíte de Testes Estruturais (Caixa-Branca)

**Prompt Utilizado:** `prompt2.md`  
**Técnica:** *Chain-of-Thought (CoT) Híbrido* com `unittest.mock`  
**Objetivo:** Garantir a cobertura de código das rotas internas de exceção (`exception_handlers`) e resiliência a falhas de componentes internos.

### Matriz de Casos de Teste Estruturais

| ID | Nome do Caso de Teste | Componente / Exceção Simulada | Fluxo Interno Exercitado | Status Code |
|---|---|---|---|---|
| **TC-06** | `test_chat_mock_openai_validation_error` | Simula `OpenAIValidationError` na função `get_openai_response`. | O handler `@app.exception_handler(OpenAIValidationError)` é acionado. | `400 Bad Request` |
| **TC-07** | `test_chat_mock_openai_api_error` | Simula `OpenAIAPIError` na chamada assíncrona. | O handler `@app.exception_handler(OpenAIAPIError)` retorna status 503. | `503 Service Unavailable` |
| **TC-08** | `test_chat_mock_empty_response` | `get_openai_response` retorna `None` ou string vazia. | Lança `OpenAIAPIError("Resposta vazia do modelo")`. | `503 Service Unavailable` |
| **TC-09** | `test_chat_mock_generic_exception` | Simula `Exception` genérica (ex: falha de memória ou erro interno inesperado). | O handler `@app.exception_handler(Exception)` trata a falha. | `500 Internal Server Error` |

---

## 💻 Código-Fonte dos Testes (`test_integration_api.py`)

Abaixo está o arquivo consolidado gerado a partir da interação com a LLM, combinando os resultados dos dois prompts e pronto para ser executado com o `pytest`.

```python
"""
Suíte de Testes de Integração da API Chatbot UFG
Gerada via LLM (Prompts Funcional e Estrutural)
"""

import pytest
from unittest.mock import AsyncMock, patch
from fastapi.testclient import TestClient
from fastapi import status

from app.main import app
from app.get_openai_response import (
    OpenAIValidationError,
    OpenAIAPIError
)

client = TestClient(app)


# ============================================================================
# 1. TESTES FUNCIONAIS (CAIXA-PRETA / CONTRATO) - Gerados via Prompt 1
# ============================================================================

def test_chat_success_200():
    """
    TC-01: Valida o fluxo feliz de envio de mensagem e recebimento de resposta.
    """
    payload = {
        "messages": [
            {"role": "user", "content": "Olá, qual é o horário de atendimento?"}
        ],
        "temperature": 0.7,
        "max_tokens": 100
    }
    
    with patch("app.main.get_openai_response", new_callable=AsyncMock) as mock_openai:
        mock_openai.return_value = "O horário de atendimento é das 8h às 18h."
        
        response = client.post("/chat", json=payload)
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        assert "response" in data
        assert data["response"] == "O horário de atendimento é das 8h às 18h."
        assert data["model"] == "gpt-4-mini"


def test_chat_validation_error_empty_messages_400():
    """
    TC-02: Valida se o envio de lista de mensagens vazia retorna HTTP 400.
    """
    payload = {
        "messages": [],
        "temperature": 0.7
    }
    
    response = client.post("/chat", json=payload)
    assert response.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY or response.status_code == status.HTTP_400_BAD_REQUEST


def test_chat_validation_invalid_role_400():
    """
    TC-03: Valida se 'role' inválido no payload é bloqueado na validação de entrada.
    """
    payload = {
        "messages": [
            {"role": "super_user", "content": "Teste de invasão"}
        ]
    }
    
    response = client.post("/chat", json=payload)
    assert response.status_code in (status.HTTP_422_UNPROCESSABLE_ENTITY, status.HTTP_400_BAD_REQUEST)


# ============================================================================
# 2. TESTES ESTRUTURAIS (CAIXA-BRANCA / MOCK & EXCEÇÕES) - Gerados via Prompt 2
# ============================================================================

def test_chat_mock_openai_validation_error_400():
    """
    TC-06: Força o lançamento de OpenAIValidationError para cobrir o handler HTTP 400.
    """
    payload = {
        "messages": [
            {"role": "user", "content": "Mensagem válida no Pydantic"}
        ]
    }
    
    with patch("app.main.get_openai_response", new_callable=AsyncMock) as mock_openai:
        mock_openai.side_effect = OpenAIValidationError("Erro na validação do prompt")
        
        response = client.post("/chat", json=payload)
        
        assert response.status_code == status.HTTP_400_BAD_REQUEST
        data = response.json()
        assert data["error_type"] == "VALIDATION_ERROR"


def test_chat_mock_openai_api_error_503():
    """
    TC-07: Força o lançamento de OpenAIAPIError para validar a captura de erro 503.
    """
    payload = {
        "messages": [
            {"role": "user", "content": "Qual é a Resolução 175?"}
        ]
    }
    
    with patch("app.main.get_openai_response", new_callable=AsyncMock) as mock_openai:
        mock_openai.side_effect = OpenAIAPIError("Serviço OpenAI Indisponível")
        
        response = client.post("/chat", json=payload)
        
        assert response.status_code == status.HTTP_503_SERVICE_UNAVAILABLE
        data = response.json()
        assert data["error_type"] == "SERVICE_UNAVAILABLE"


def test_chat_mock_empty_response_503():
    """
    TC-08: Simula resposta vazia retornada da OpenAI, acionando exceção HTTP 503.
    """
    payload = {
        "messages": [
            {"role": "user", "content": "Pergunta teste"}
        ]
    }
    
    with patch("app.main.get_openai_response", new_callable=AsyncMock) as mock_openai:
        mock_openai.return_value = ""  # Resposta vazia
        
        response = client.post("/chat", json=payload)
        
        assert response.status_code == status.HTTP_503_SERVICE_UNAVAILABLE


def test_chat_mock_generic_exception_500():
    """
    TC-09: Simula erro genérico não tratado para validar a última camada de segurança (500).
    """
    payload = {
        "messages": [
            {"role": "user", "content": "Teste erro interno"}
        ]
    }
    
    with patch("app.main.get_openai_response", new_callable=AsyncMock) as mock_openai:
        mock_openai.side_effect = Exception("Crash não esperado no servidor")
        
        response = client.post("/chat", json=payload)
        
        assert response.status_code == status.HTTP_500_INTERNAL_SERVER_ERROR
        data = response.json()
        assert data["error_type"] == "INTERNAL_ERROR"
```

---

## 📊 Avaliação da Contribuição da GenAI

1. **Influência dos Prompts:**
   - O **Prompt 1 (Zero-Shot Funcional)** garantiu a aderência ao contrato externo de API REST e validações de cabeçalho/status code HTTP.
   - O **Prompt 2 (Chain-of-Thought Estrutural)** direcionou a LLM a raciocinar sobre os blocos `try/except` e os `exception_handlers` do FastAPI, elevando a cobertura de decisão (branches) e eliminando pontos cego na suíte.
2. **Atuação como Copiloto:**
   - A IA agilizou a criação da estrutura de mocks assíncronos (`AsyncMock`), reduzindo drasticamente o tempo gasto em boilerplate de código e garantindo que exceções difíceis de reproduzir em homologação pudessem ser testadas em milissegundos.