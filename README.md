# Streaming Text Embeddings for Retrieval Augmented Generation (RAG)

How to use Redpanda and Benthos to generate vector embeddings on streaming text.

<p align="center">
    <img src="./embeddings.png" width="75%">
</p>

Retrieval Augmented Generation (RAG) is best described by [OpenAI](https://help.openai.com/en/articles/8868588-retrieval-augmented-generation-rag-and-semantic-search-for-gpts) as _"the process of retrieving relevant contextual information from a data source and passing that information to a large language model alongside the user's prompt. This information is used to improve the model's output by augmenting the model's base knowledge"._

RAG comprises of two parts:

1. The acquisition and persistence of new information that the model has no prior knowledge of. This new information (documents, webpages, etc) is split into small chunks of text and stored in a vector database along with its vector embeddings. This is the bit we're going to do in near realtime using Redpanda.

> [!NOTE]
> Vector embeddings are a mathematical representation of text that encode its semantic meaning in a vector space. Words with a similar meaning will be clustered closer together in this multidimensional space. Vector databases, such as [MongoDB Atlas](https://www.mongodb.com/products/platform/atlas-vector-search), have the ability to query vector embeddings to retrieve texts with a similar semantic meaning. A famous example of this being: `king - man + woman = queen`.

2. The retrieval of relevant contextual information from a vector database (semantic search), and the passing of that information alongside a user's question (prompt) to the large language model to improve the model's generated answer.

In this Lab, we'll demonstrate how to build a stream processing pipeline using the Redpanda Platform to add [OpenAI text embeddings](https://platform.openai.com/docs/guides/embeddings) to messages as they stream through Redpanda on their way to a MongoDB Atlas vector database. As part of the solution we'll also use the [LangChain](https://www.langchain.com/) framework to acquire and send new information to Redpanda, and to build the prompt that retrieves relevant texts from the vector database and uses that context to query OpenAI's `gpt-4o` model.

## Prerequisites

### Redpanda Serverless

Log in to your Redpanda Cloud account (or sign up if you don't have one) and create a new [Serverless](https://redpanda.com/redpanda-cloud/serverless) cluster. Make a note of the bootstrap server URL, and create a new user with permissions (ACLs) to access a topic named `documents` and a consumer group named `benthos`. Add the cluster connection information to a local `.env` file:

```bash
% cat > demo/.env<< EOF
REDPANDA_SERVERS="<bootstrap_server_url>"
REDPANDA_USER="<username>"
REDPANDA_PASS="<password>"
REDPANDA_TOPICS="documents"

EOF
```

### OpenAI API

Log in to your OpenAI developer platform account (or sign up if you don't have one) and create a new [Project API key](https://platform.openai.com/api-keys). Add the secret key to the local `.env` file:

```bash
% cat >> demo/.env<< EOF
OPENAI_API_KEY="<secret_key>"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
OPENAI_MODEL="gpt-4o"

EOF
```

### MongoDB Atlas

Log in to your MongoDB Atlas account (or sign up if you don't have one) and deploy a new [free cluster](https://www.mongodb.com/docs/atlas/getting-started) for development purposes. Create a new database named `VectorStore`, a new collection in that database named `Embeddings`, and an Atlas Vector Search index with the following JSON configuration:

```json
{
  "fields": [
    {
      "numDimensions": 1536, //text-embedding-3-small
      "path": "embedding",
      "similarity": "euclidean",
      "type": "vector"
    }
  ]
}
```

Add the Atlas connection information to the local `.env` file:

```bash
% cat >> demo/.env<< EOF
# Connection string for MongoDB Driver for Go:
ATLAS_CONNECTION_STRING="mongodb+srv://<username>:<password>@vectorstore.ozmdcxv.mongodb.net/?retryWrites=false"
ATLAS_DB="VectorStore"
ATLAS_COLLECTION="Embeddings"
ATLAS_INDEX="vector_index"

EOF
```

### Set environment variables

If all of the prerequisites have been completed then the `.env` file should look like this:

```bash
% cat demo/.env
REDPANDA_SERVERS="<bootstrap_server_url>"
REDPANDA_USER="<username>"
REDPANDA_PASS="<password>"
REDPANDA_TOPICS="documents"

OPENAI_API_KEY="<secret_key>"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
OPENAI_MODEL="gpt-4o"

ATLAS_CONNECTION_STRING="mongodb+srv://<username>:<password>@vectorstore.ozmdcxv.mongodb.net/?retryWrites=false"
ATLAS_DB="VectorStore"
ATLAS_COLLECTION="Embeddings"
ATLAS_INDEX="vector_index"
```

The Python files in the `demo` directory will load these variables directly from the `.env` file, but you also need to set them in the environment for Benthos configuration interpolation:

```bash
% export $(grep -v '^#' ./demo/.env | xargs)
```

### Create Python virtual environment

Last but not least, create the Python virtual environment in the `demo` directory:

```bash
% cd demo
% python3 -m venv env
% source env/bin/activate
% pip install -r requirements.txt
```

## Run the lab

The Lab comprises of three parts:

1. Use *LangChain's* `WebBaseLoader` and `RecursiveCharacterTextSplitter` to generate chunks of text from the BBC Sport website and send each chunk to a Redpanda topic named `documents`.
2. Use *Redpanda Connect* to consume the messages from the `documents` topic and pass each message through a custom processor that calls *OpenAI's embeddings API* to retrieve the vector embeddings for the text. The enriched messages are then inserted into a *MongoDB Atlas* database collection that has a vector search index.
3. Complete the RAG pipeline by using *LangChain* to retrieve similar texts from the *MongoDB Atlas* database and add that context alongside a user question to a prompt that is sent to OpenAI's new `gpt-4o` model.

### Start Benthos 

```bash
#
# Terminal 1. Start Benthos with custom OpenAI processor.
#
% go test
PASS
ok  	benthos-embeddings	2.472s

% go build
% export $(grep -v '^#' ./demo/.env | xargs)
% ./benthos-embeddings -c demo/rag_demo.yaml --log.level debug

INFO Running main config from specified file       @service=benthos benthos_version=v4.27.0 path=demo/rag_demo.yaml
INFO Listening for HTTP requests at: http://0.0.0.0:4195  @service=benthos
DEBU url: https://api.openai.com/v1/embeddings, model: text-embedding-3-small  @service=benthos label="" path=root.pipeline.processors.0
INFO Launching a benthos instance, use CTRL+C to close  @service=benthos
INFO Input type kafka is now active                @service=benthos label="" path=root.input
DEBU Starting consumer group                       @service=benthos label="" path=root.input
INFO Output type mongodb is now active             @service=benthos label="" path=root.output
```

### Generate new text documents

```bash
#
# Terminal 2. Generate new text documents and send them to Atlas, via Redpanda Connect for embeddings.
#             You can view the text and embeddings in the Atlas console: https://cloud.mongodb.com. 
% cd demo
% source env/bin/activate
# Single webpage:
% python produce_documents.py -u "https://www.bbc.co.uk/sport/football/articles/c3gglr8mpzdo"
# Entire sitemap:
% python produce_documents.py -s "https://www.bbc.com/sport/sitemap.xml"
```

### Run the retrieval and generation chain

```bash
#
# Terminal 3. Run the retrieval chain and ask OpenAI a question.
#
% cd demo
% source env/bin/activate
% python retrieve_generate.py -q """
  Which football players made the provisional England national squad for the Euro 2024 tournament,
  and on what date was this announced?
  """

Question: 

Which football players made the provisional England national squad for the Euro 2024 tournament, and on what date was this announced?

Initial answer: 

As of my knowledge cutoff date in October 2023, the provisional England national squad for the Euro 2024 tournament has not been announced. The selection of national teams for major tournaments like the UEFA European Championship typically happens closer to the event, often just a few weeks before the tournament starts. For the most current information, I recommend checking the latest updates from the Football Association (FA) or other reliable sports news sources.

Augmented answer: 

The provisional England national squad for the Euro 2024 tournament includes the following players:

**Goalkeepers:**
- Dean Henderson (Crystal Palace)
- Jordan Pickford (Everton)
- Aaron Ramsdale (Arsenal)
- James Trafford (Burnley)

**Defenders:**
- Jarrad Branthwaite (Everton)
- Lewis Dunk (Brighton)
- Joe Gomez (Liverpool)
- Marc Guehi (Crystal Palace)
- Ezri Konsa (Aston Villa)
- Harry Maguire (Manchester United)
- Jarell Quansah (Liverpool)
- Luke Shaw (Manchester United)
- John Stones (Manchester City)
- Kieran Trippier (Newcastle)
- Kyle Walker (Manchester City)

**Midfielders:**
- Trent Alexander-Arnold (Liverpool)
- Conor Gallagher (Chelsea)
- Curtis Jones (Liverpool)
- Kobbie Mainoo (Manchester United)
- Declan Rice (Arsenal)
- Adam Wharton (Crystal Palace)

**Forwards:**
- Jude Bellingham (Real Madrid)
- Jarrod Bowen (West Ham)
- Eberechi Eze (Crystal Palace)
- Phil Foden (Manchester City)
- Jack Grealish (Manchester City)
- Anthony Gordon (Newcastle)
- Harry Kane (Bayern Munich)
- James Maddison (Tottenham)
- Cole Palmer (Chelsea)
- Bukayo Saka (Arsenal)
- Ivan Toney (Brentford)
- Ollie Watkins (Aston Villa)

This announcement was made on May 21, 2024.
```
