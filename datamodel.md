## Mapeamento de Entidades do Sistema
## Entidade 1: DocumentoChunk (Trecho do Regimento)
Finalidade: Representa uma unidade segmentada de texto extraída do Regimento Acadêmico ou Manual do Aluno, contendo o conteúdo textual, seus vetores de embedding e os metadados de localização original para fundamentação do RAG.
Campos e Significados:
-id: Identificador único e estável do trecho. Obrigatório: sim. Regra: texto único, gerado via hash do conteúdo e localização (ex: doc_regimento_art45_p12).
-conteudo_texto: Texto bruto extraído do trecho do PDF. Obrigatório: sim. Regra: string não vazia com tamanho entre 200 e 1000 caracteres.
-documento_fonte: Nome do arquivo de origem. Obrigatório: sim. Regra: ex: Regimento_Academico_2026.pdf.
-capitulo: Nome do capítulo ou título correspondente. Obrigatório: não. Regra: string descritiva.
-artigo: Número do artigo ou seção identificada. Obrigatório: sim. Regra: texto estruturado (ex: Art. 45 ou Seção II).
-pagina: Número da página original no PDF. Obrigatório: sim. Regra: número inteiro maior ou igual a 1.
-vetor_embedding: Lista de números de alta dimensão representando o significado semântico do texto. Obrigatório: sim. Regra: vetor numérico de dimensão compatível com o modelo de embeddings do ChromaDB.

## Relacionamentos:
DocumentoChunk possui relação de pertinência com a busca RAG acionada na entidade Mensagem.
Estados e transições:
NaoProcessado -> Indexado -> Obsoleto
A transição de NaoProcessado para Indexado ocorre quando o script de ingestão fatiar o PDF, gerar os embeddings e salvar com sucesso no ChromaDB.
A transição de Indexado para Obsoleto ocorre quando uma nova versão do PDF do Regimento for carregada e substituir a coleção no banco vetorial.
A transição é rejeitada quando o arquivo PDF de origem estiver corrompido, sem camada de texto legível ou com metadados de página ausentes.

## Entidade 2: Mensagem (Interação de Chat)
Finalidade: Representa um turno individual de comunicação dentro do chat, registrando o texto enviado pelo aluno, a resposta gerada pelo assistente, as fontes citadas e os metadados de execução.
## Campos e Significados:
id: Identificador único da mensagem. Obrigatório: sim. Regra: UUID v4 gerado automaticamente.
sessao_id: Identificador da sessão ativa do usuário. Obrigatório: sim. Regra: string única vinculada à conversa ativa.
papel: Papel do emissor da mensagem. Obrigatório: sim. Regra: deve ser estritamente user, assistant ou system.
conteudo: Texto da mensagem enviada ou recebida. Obrigatório: sim. Regra: string não vazia.
fontes_citadas: Lista de referências recuperadas do RAG utilizadas na resposta. Obrigatório: não. Regra: lista de objetos contendo artigo e pagina (preenchido apenas quando o papel for assistant).
ferramenta_acionada: Nome da Tool executada no turno. Obrigatório: não. Regra: calcular_nota_final ou nulo.
timestamp: Data e hora de registro da mensagem. Obrigatório: sim. Regra: data/hora em padrão UTC/ISO-8601.
## Relacionamentos:
SessaoConversa possui uma coleção de várias Mensagens.
Mensagem (quando do tipo assistente) possui referências com múltiplos DocumentoChunks citados.
## Estados e transições:
Recebida -> EmProcessamento -> Respondida (ou Falhou)
A transição para EmProcessamento ocorre no momento em que a mensagem do usuário chega ao endpoint /api/v1/chat do FastAPI.
A transição para Respondida ocorre quando o Agente sintetiza a resposta final com RAG ou com o resultado da Tool.
A transição para Falhou ocorre quando a API de IA expira (timeout de 6s) ou se o serviço externo falhar.
A transição é rejeitada se o payload da requisição estiver vazio ou violar a validação do schema Pydantic.

## Entidade 3: ResultadoCalculo (Resultado da Tool Determinística)
Finalidade: Representa a estrutura de dados gerada exclusivamente pela função Python calcular_nota_final(p1, p2), garantindo o determinismo e a precisão na verificação de aprovação do estudante.

## Campos e Significados:
p1: Nota da primeira avaliação parcial. Obrigatório: sim. Regra: número decimal (float) com valor entre 0.0 e 10.0.
p2: Nota da segunda avaliação parcial. Obrigatório: sim. Regra: número decimal (float) com valor entre 0.0 e 10.0.
media_parcial: Média aritmética simples entre P1 e P2. Obrigatório: sim. Regra: valor calculado por (p1 + p2) / 2.
status: Situação acadêmica parcial resultante. Obrigatório: sim. Regra: valor estrito entre Aprovado Direto, Em Prova Final ou Reprovado Direto.
nota_necessaria_final: Nota mínima exigida no exame final. Obrigatório: não. Regra: valor calculado por 12 - media_parcial quando o status for Em Prova Final; caso contrário, valor nulo.
## Relacionamentos:
Mensagem (do assistente) possui um ResultadoCalculo associado quando a intenção de cálculo é identificada.
## Estados e transições:
Solititado -> Validado -> Calculado
A transição de Solicitado para Validado ocorre quando o agente extrai os parâmetros p1 e p2 da mensagem e os valida via schema Pydantic.
A transição para Calculado ocorre no momento da execução determinística da função Python.
A transição é rejeitada quando p1 ou p2 contiverem valores fora do intervalo de 0.0 a 10.0 ou tipos não numéricos.

## Validações Transversais
Regra entre Nota e Status Acadêmico: Se a media_parcial for menor que 4.0, o status deve ser obrigatoriamente Reprovado Direto e a nota_necessaria_final deve ser nula. Se a media_parcial for maior ou igual a 7.0, o status deve ser Aprovado Direto e a nota_necessaria_final deve ser nula.
Integridade de Citação do RAG: Toda resposta gerada pelo assistente baseada em normas do regimento deve obrigatoriamente mapear pelo menos um DocumentoChunk existente contendo artigo e pagina válidos.
Privacidade e Retenção de Dados: Nenhuma informação de identificação pessoal (PII) dos alunos é armazenada. O histórico de mensagens na entidade Mensagem é mantido apenas em memória durante a sessão ativa do usuário.
Segurança na Invocação de Tools: A execução do cálculo na entidade ResultadoCalculo nunca deve ser realizada por interpretação de texto do LLM, sendo restrita à invocação de código Python puro através do mecanismo de Function Calling.