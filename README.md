Chatbot UFG - Testes de Integração com IA Generativa
Este repositório contém a suíte de testes de integração para a API REST do Chatbot UFG, focando na integração entre o endpoint /chat, o middleware de segurança e o serviço OpenAI.
🎯 Objetivo
Avaliar e demonstrar o uso de Inteligência Artificial Generativa (LLMs) como "copiloto" na criação de testes de integração robustos, aplicando técnicas de Prompt Engineering estruturadas.
🚀 Como Configurar e Executar
Clone o repositório:

git clone https://github.com/senisem/ESP-T1-C13-_-API-TESTES

cd ESP-T1-C13-_-API-TESTES

Crie um ambiente virtual e instale as dependências:

python -m venv venv

source venv/bin/activate  # No Windows: venv\Scripts\activate

pip install pytest httpx pytest-asyncio

Execute os testes:

pytest test_integration_api.py -v
🧠 Estratégia de Prompts (Prompt Engineering)
Neste projeto, utilizamos diferentes abordagens para guiar o LLM na geração dos testes:

Testes Funcionais (Zero-Shot Estruturado): Foco no contrato da API (Caixa Preta). Definimos o papel do LLM como testador Sênior e listamos os Status Codes (200, 400, 429, 503) e as regras de negócio a serem verificadas.
Testes Estruturais (Chain-of-Thought Híbrido): Foco na cobertura de código (Caixa Branca). Injetamos trechos do código-fonte (ex: main.py) e instruímos o LLM a pensar passo a passo para mapear as exceções internas (OpenAIAPIError) para as respostas HTTP corretas, utilizando unittest.mock.
