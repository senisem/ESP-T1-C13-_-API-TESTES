Você é um Engenheiro de Testes Sênior especializado em cobertura de código e testes de integração. Sua tarefa é criar testes white-box em `pytest` utilizando `unittest.mock` para simular as respostas da função `get_openai_response`.

Pense passo a passo (Chain-of-Thought):
1. Identifique os fluxos de exceção no endpoint `/chat` (OpenAIValidationError, OpenAIAPIError, Exception genérica).
2. Para cada regra, crie um cenário onde o mock da OpenAI lança a exceção correspondente.
3. Estruture os testes verificando se o retorno da API FastAPI mapeia corretamente para os status 400, 503 e 500.

Contexto do código-fonte delimitado por #####:
#####
@app.exception_handler(OpenAIValidationError)
async def openai_validation_error_handler(request, exc: OpenAIValidationError):
    """Handler para erros de validação do OpenAI."""
    logger.warning(f"Erro de validação: {exc}")
    return JSONResponse(
        status_code=status.HTTP_400_BAD_REQUEST,
        content=ErrorResponse(
            status_code=400,
            message="Erro de validação na requisição",
            error_type="VALIDATION_ERROR",
            detail=str(exc)
        ).model_dump()
    )


@app.exception_handler(OpenAIAPIError)
async def openai_api_error_handler(request, exc: OpenAIAPIError):
    """Handler para erros da API OpenAI."""
    logger.error(f"Erro da API OpenAI: {exc}")
    return JSONResponse(
        status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
        content=ErrorResponse(
            status_code=503,
            message="Serviço OpenAI indisponível",
            error_type="SERVICE_UNAVAILABLE",
            detail="Não foi possível processar sua requisição. Tente novamente mais tarde."
        ).model_dump()
    )


@app.exception_handler(OpenAIError)
async def openai_error_handler(request, exc: OpenAIError):
    """Handler genérico para erros OpenAI."""
    logger.error(f"Erro OpenAI não esperado: {exc}")
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content=ErrorResponse(
            status_code=500,
            message="Erro interno do servidor",
            error_type="INTERNAL_ERROR",
            detail="Ocorreu um erro inesperado. Contate o administrador."
        ).model_dump()
    )


@app.exception_handler(Exception)
async def generic_exception_handler(request, exc: Exception):
    """Handler genérico para exceções não tratadas."""
    logger.error(f"Erro inesperado: {type(exc).__name__}: {exc}", exc_info=True)
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content=ErrorResponse(
            status_code=500,
            message="Erro interno do servidor",
            error_type="INTERNAL_ERROR",
            detail="Ocorreu um erro inesperado."
        ).model_dump()
    )


@app.post(
    "/chat",
    response_model=ChatResponse,
    status_code=status.HTTP_200_OK,
    tags=["Chat"],
    summary="Enviar mensagem para o Chatbot",
    description="Processa uma mensagem de chat e retorna a resposta do modelo OpenAI",
    responses={
        200: {"description": "Resposta obtida com sucesso"},
        400: {"description": "Erro de validação na requisição"},
        429: {"description": "Limite de requisições excedido"},
        503: {"description": "Serviço OpenAI indisponível"}
    }
)
async def chat_endpoint(
    request: ChatRequest,
    settings: Settings = Depends(get_settings)
) -> ChatResponse:
    """
    Endpoint principal do chatbot.
    """
    
    try:
        logger.info(
            f"Requisição de chat recebida com {len(request.messages)} mensagens"
        )
        
        # Converter mensagens Pydantic para dicionários
        messages_dict = [msg.model_dump() for msg in request.messages]
        
        # Criar configuração do OpenAI
        openai_config = OpenAIConfig(
            api_key=settings.openai_api_key,
            model=settings.openai_model,
            temperature=request.temperature or settings.openai_temperature,
            max_tokens=request.max_tokens or settings.openai_max_tokens,
            timeout=settings.openai_timeout,
            max_retries=settings.openai_max_retries
        )
        
        # Chamar função assíncrona de OpenAI
        logger.debug(f"Chamando OpenAI com modelo: {openai_config.model}")
        response_text = await get_openai_response(
            messages=messages_dict,
            config=openai_config
        )
        
        if not response_text:
            logger.error("OpenAI retornou resposta vazia")
            raise OpenAIAPIError("Resposta vazia do modelo")
        
        logger.info("Resposta OpenAI obtida com sucesso")
        
        return ChatResponse(
            response=response_text,
            model=settings.openai_model,
            usage={
                "model": settings.openai_model,
                "timestamp": str(__import__('datetime').datetime.now()),
                "temperature": request.temperature or settings.openai_temperature,
                "max_tokens": request.max_tokens or settings.openai_max_tokens
            }
        )
    
    except OpenAIValidationError as e:
        logger.warning(f"Erro de validação: {e}")
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )
    
    except OpenAIAPIError as e:
        logger.error(f"Erro da API OpenAI: {e}")
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="Serviço OpenAI indisponível"
        )
    
    except Exception as e:
        logger.error(f"Erro inesperado no endpoint /chat: {e}", exc_info=True)
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Erro interno do servidor"
        )
#####
Forneça apenas o código Python final dos testes.