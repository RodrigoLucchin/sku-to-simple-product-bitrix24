# Projeto de Sincronização de Catálogo com n8n, Supabase e Bitrix24

Este projeto contém uma suíte de três workflows do n8n projetados para realizar uma operação completa de **ETL (Extract, Transform, Load)** e gerenciamento de produtos (SKUs) entre um sistema de catálogo de origem e um catálogo de destino no **Bitrix24**.

A solução utiliza um banco de dados **Supabase** como uma área de *staging* (intermediária), o que permite desacoplar o processo de extração do processo de carregamento, oferecendo maior controle, resiliência e capacidade de auditoria.

##  Tecnologias Utilizadas

* **n8n:** A plataforma de automação que orquestra todo o fluxo de dados.
* **Bitrix24 API:** A interface para interagir com o catálogo de produtos de destino, tanto para criação quanto para exclusão.
* **Supabase (PostgreSQL):** Atua como banco de dados intermediário para armazenar, gerenciar e rastrear o estado de sincronização de cada produto.

##  Visão Geral e Ordem de Execução

O projeto é dividido em três workflows distintos, que devem ser executados em uma sequência específica para garantir o funcionamento correto do processo.

---

### **Passo 1: `1. Add SKU Database`**

Este é o primeiro workflow do processo, responsável pela **Extração e Transformação (ET)** dos dados.

* **Gatilho:** **Manual**.
* **Funcionalidade Principal:**
    1.  **Extração:** Conecta-se à API de um catálogo de origem para buscar a lista de produtos (SKUs), lidando com a paginação para obter todos os itens.
    2.  **Enriquecimento:** Itera sobre cada produto individualmente para buscar dados adicionais e detalhados, como preços, imagens e descrições completas, através de chamadas de API complementares.
    3.  **Transformação:** Estrutura e limpa todos os dados coletados em um formato padronizado.
    4.  **Carregamento (Staging):** Insere os dados de produtos formatados em uma tabela no **Supabase**. Esta tabela servirá como a fonte de dados para o próximo passo.

---

### **Passo 2: `2. Criar Itens Bitrix`**

Este workflow é responsável pela **Carga (Load)** dos dados no sistema de destino.

* **Gatilho:** **Manual**.
* **Funcionalidade Principal:**
    1.  **Leitura do Staging:** Consulta o banco de dados Supabase para buscar os produtos que foram inseridos no Passo 1, mas que ainda não foram criados no Bitrix24 (identificados por uma flag, ex: `created_new IS NULL`).
    2.  **Processamento de Imagens:** Verifica se o produto possui uma imagem. Se possuir, o workflow faz o download do arquivo e o converte para o formato **Base64**, que é o padrão exigido pela API do Bitrix24.
    3.  **Criação no Catálogo:** Itera sobre cada produto e utiliza a API do Bitrix24 (`catalog.product.add`) para criar o item no catálogo de destino, enviando todos os seus detalhes (incluindo a imagem, se houver).
    4.  **Atualização de Status:** Após criar o produto e adicionar seu preço com sucesso, o workflow atualiza o registro correspondente no Supabase (ex: `SET created_new = 'Y'`), marcando-o como "concluído" para evitar duplicações futuras.

---

### **Passo 3: `3. Deletar Todos SKU`**

Este é um workflow de **manutenção e limpeza**, que pode ser usado para resetar o catálogo.

* **Gatilho:** **Manual**.
* **Funcionalidade Principal:**
    1.  **Leitura dos Itens Criados:** Consulta o Supabase e busca todos os produtos que foram marcados como criados com sucesso no Bitrix24 (ex: `created_new = 'Y'`).
    2.  **Exclusão em Lote:** Itera sobre a lista de produtos e faz chamadas à API do Bitrix24 (`catalog.product.delete`) para remover cada um deles do catálogo.
    3.  **Atualização de Status:** Após a exclusão, atualiza o registro no Supabase (ex: `SET deleted = 'Y'`) para registrar que a ação foi concluída.

##  Arquitetura e Lógica Central

A utilização do **Supabase como um banco de dados de staging** é o pilar desta arquitetura. Ele funciona como uma "fila" persistente e um livro de registros, permitindo que:
* O processo de extração (Passo 1) possa ser executado independentemente da criação (Passo 2).
* Se a criação de um item falhar, ele permanecerá no banco como "não criado" e poderá ser reprocessado sem a necessidade de extrair todos os dados novamente.
* Haja um registro claro de quais produtos foram criados, quando e se já foram deletados, facilitando a auditoria e a manutenção.

##  Como Utilizar

1.  **Configuração:** Garanta que as credenciais do Supabase e da API do Bitrix24 estão devidamente configuradas nos nós correspondentes em cada workflow.
2.  **Execução:** Execute os workflows na ordem numérica designada:
    * Primeiro, execute o **`1. Add SKU Database`** para popular o banco de dados.
    * Em seguida, execute o **`2. Criar Itens Bitrix`** para criar os produtos no catálogo.
    * Se necessário, execute o **`3. Deletar Todos SKU`** para limpar os itens criados do catálogo do Bitrix24.
