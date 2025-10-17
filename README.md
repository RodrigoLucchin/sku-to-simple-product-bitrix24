# PT/BR

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

---

# EN/US

# Catalog Synchronization Project with n8n, Supabase and Bitrix24

This project contains a suite of three n8n workflows designed to perform a complete **ETL (Extract, Transform, Load)** and product (SKU) management operation between a source catalog system and a target catalog in **Bitrix24**.

The solution uses a **Supabase** database as a *staging* (intermediate) area, which allows the extraction process to be decoupled from the loading process, offering greater control, resilience and auditability.

## Technologies Used

* **n8n:** The automation platform that orchestrates the entire data flow.
* **Bitrix24 API:** The interface for interacting with the target product catalog, both for creation and deletion.
* **Supabase (PostgreSQL):** Acts as an intermediate database to store, manage and track the synchronization state of each product.

## Overview and Execution Order

The project is divided into three distinct workflows, which must be executed in a specific sequence to ensure the process functions correctly.

---

### **Step 1: `1. Add SKU Database`**

This is the first workflow of the process, responsible for **Extraction and Transformation (ET)** of data.

* **Trigger:** **Manual**.
* **Main Functionality:**
    1. **Extraction:** Connects to a source catalog's API to fetch the list of products (SKUs), handling pagination to get all items.
    2. **Enrichment:** Iterates over each individual product to fetch additional, detailed data, such as prices, images, and full descriptions, through complementary API calls.
    3. **Transformation:** Structures and cleans all collected data into a standardized format.
    4. **Staging:** Inserts formatted product data into a table in **Supabase**. This table will serve as the data source for the next step.

---

### **Step 2: `2. Create Bitrix`** Items

This workflow is responsible for **Loading** data into the target system.

* **Trigger:** **Manual**.
* **Main Functionality:**
    1. **Staging Reading:** Query the Supabase database to search for products that were inserted in Step 1, but that have not yet been created in Bitrix24 (identified by a flag, e.g. `created_new IS NULL`).
    2. **Image Processing:** Checks if the product has an image. If so, the workflow downloads the file and converts it to **Base64** format, which is the standard required by the Bitrix24 API.
    3. **Catalog Creation:** Iterates over each product and uses the Bitrix24 API (`catalog.product.add`) to create the item in the target catalog, sending all its details (including the image, if any).
    4. **Status Update:** After successfully creating the product and adding its price, the workflow updates the corresponding record in Supabase (ex: `SET created_new = 'Y'`), marking it as "completed" to avoid future duplications.

---

### **Step 3: `3. Delete All SKU`**

This is a **maintenance and cleaning** workflow, which can be used to reset the catalog.

* **Trigger:** **Manual**.
* **Main Functionality:**
    1. **Reading Created Items:** Query Supabase and search for all products that were marked as successfully created in Bitrix24 (ex: `created_new = 'Y'`).
    2. **Batch Delete:** Iterates over the list of products and makes calls to the Bitrix24 API (`catalog.product.delete`) to remove each one from the catalog.
    3. **Status Update:** After deletion, update the record in Supabase (ex: `SET deleted = 'Y'`) to record that the action was completed.

## Architecture and Core Logic

The use of **Supabase as a staging database** is the cornerstone of this architecture. It functions as a persistent "queue" and ledger, allowing you to:
* The extraction process (Step 1) can be performed independently of creation (Step 2).
* If the creation of an item fails, it will remain in the database as "not created" and can be reprocessed without having to extract all the data again.
* There is a clear record of which products were created, when and if they were deleted, facilitating auditing and maintenance.

## How to Use

1. **Configuration:** Ensure that Supabase and Bitrix24 API credentials are properly configured on the corresponding nodes in each workflow.
2. **Execution:** Execute the workflows in the designated numerical order:
    * First, run **`1. Add SKU Database`** to populate the database.
    * Then run **`2. Create Bitrix`** Items to create the products in the catalog.
    * If necessary, run **`3. Delete All SKU`** to clear the created items from the Bitrix24 catalog.
