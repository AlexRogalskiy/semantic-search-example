# Semantic Search

In this walkthrough we will see how to use Pinecone for semantic search. To begin we must install the required prerequisite libraries:

## Setup

Ensure you have Node.js and `npm` installed. Clone the repository and install the dependencies using `npm install`.

### Configuration

Create an `.env` file in the root of the project and add your Pinecone API key and environment details:

```sh
PINECONE_API_KEY=<your-api-key>
PINECONE_ENVIRONMENT=<your-environment>
PINECONE_INDEX=<index-name>
```

## Application structure

There are two main components to this application: the data loader and the search engine. The data loader is responsible for loading the data into Pinecone. The search engine is responsible for querying the index and returning the results. These two components share two common modules: the `embedder` and the `pinecone` utility module.

## Data Preprocessing

The data loading process starts with the CSV file. This file contains the articles that will be indexed and made searchable. To load this data, the project uses the `papaparse` library. The loadCSVFile function in `csvLoader.ts` reads the file and uses `papaparse` to parse the CSV data into JavaScript objects. The `dynamicTyping` option is set to true to automatically convert the data to the appropriate types. After this step, you will have an array of objects, where each object represents an article​.

```typescript
import fs from "fs/promises";
import Papa from "papaparse";

async function loadCSVFile(
  filePath: string
): Promise<Papa.ParseResult<Record<string, unknown>>> {
  try {
    // Get csv file absolute path
    const csvAbsolutePath = await fs.realpath(filePath);

    // Create a readable stream from the CSV file
    const data = await fs.readFile(csvAbsolutePath, "utf8");

    // Parse the CSV file
    return await Papa.parse(data, {
      dynamicTyping: true,
      header: true,
      skipEmptyLines: true,
    });
  } catch (err) {
    console.error(err);
    throw err;
  }
}

export { loadCSVFile };
```

## Building embeddings

The text embedding operation is performed in the `Embedder` class. This class uses a pipeline from the [`@xenova/transformers`](https://github.com/xenova/transformers.js) library to generate embeddings for the input text. We use the [`sentence-transformers/all-MiniLM-L6-v2`](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2) model to generate the embeddings. The class provides methods to embed a single string, an array of strings, or an array of strings in batches​ - which will come in useful a bit later.

```typescript
import { Vector } from "@pinecone-database/pinecone";
import { Pipeline, pipeline } from "@xenova/transformers";
import { v4 as uuidv4 } from "uuid";
import { sliceIntoChunks } from "./utils/util";

class Embedder {
  private pipe: Pipeline | null = null;

  // Initialize the pipeline
  async init() {
    this.pipe = await pipeline(
      "embeddings",
      "sentence-transformers/all-MiniLM-L6-v2"
    );
  }

  // Embed a single string
  async embed(text: string): Promise<Vector> {
    const result = this.pipe && (await this.pipe(text));
    return {
      id: uuidv4(),
      metadata: {
        text,
      },
      values: Array.from(result.data),
    };
  }

  // Batch an array of string and embed each batch
  // Call onDoneBatch with the embeddings of each batch
  async embedBatch(
    texts: string[],
    batchSize: number,
    onDoneBatch: (embeddings: Vector[]) => void
  ) {
    const batches = sliceIntoChunks<string>(texts, batchSize);
    for (const batch of batches) {
      const embeddings = await Promise.all(
        batch.map((text) => this.embed(text))
      );
      await onDoneBatch(embeddings);
    }
  }
}

const embedder = new Embedder();

export { embedder };
```

## Pinecone utility function

This function ensures that the required environment variables are set, and then initializes the Pinecone client. To save unnecessary instantiations of the Pinecone client, we use a singleton pattern to ensure that only one instance of the client is created.

```typescript
import { PineconeClient } from "@pinecone-database/pinecone";
import { config } from "dotenv";
import { getEnv, validateEnvironmentVariables } from "./utils/util";

config();

let pineconeClient: PineconeClient | null = null;

// Returns a Promise that resolves to a PineconeClient instance
export const getPineconeClient = async (): Promise<PineconeClient> => {
  validateEnvironmentVariables();

  if (pineconeClient) {
    return pineconeClient;
  } else {
    pineconeClient = new PineconeClient();

    await pineconeClient.init({
      apiKey: getEnv("PINECONE_API_KEY"),
      environment: getEnv("PINECONE_ENVIRONMENT"),
    });
  }
  return pineconeClient;
};
```

## Indexing

Now that we have a way to load data and create embeddings, let put the two together and save the embeddings in Pinecone. In the following section, we get the path of the file we need to process from the command like. We load the CSV file, create the Pinecone index and then start the embedding process. The embedding process is done in batches of 1000. Once we have a batch of embeddings, we insert them into the index.

```typescript
import { utils } from "@pinecone-database/pinecone";
import cliProgress from "cli-progress";
import { config } from "dotenv";
import fs from "fs";
import { loadCSVFile } from "./csvLoader";
import { embedder } from "./embeddings";
import { getPineconeClient } from "./pinecone";
import { getEnv, getIndexingCommandLineArguments } from "./utils/util";
const { createIndexIfNotExists, chunkedUpsert } = utils;

config();

const progressBar = new cliProgress.SingleBar(
  {},
  cliProgress.Presets.shades_classic
);
const indexName = getEnv("PINECONE_INDEX");
let counter = 0;

const run = async () => {
  // Get the CSV path and column name from the command line arguments
  const { csvPath, column } = getIndexingCommandLineArguments();

  // Get a PineconeClient instance
  const pineconeClient = await getPineconeClient();

  // Get csv file absolute path
  const csvAbsolutePath = fs.realpathSync(csvPath);

  // Create a readable stream from the CSV file
  const { data, meta } = await loadCSVFile(csvAbsolutePath);

  // Ensure the selected column exists in the CSV file
  if (!meta.fields?.includes(column)) {
    console.error(`Column ${column} not found in CSV file`);
    process.exit(1);
  }

  // Extract the selected column from the CSV file
  const documents = data.map((row) => row[column] as string);

  // Create a Pinecone index with the name "word-embeddings" and a dimension of 384
  await createIndexIfNotExists(pineconeClient, indexName, 384);

  // Select the target Pinecone index
  const index = pineconeClient.Index(indexName);

  // Start the progress bar
  progressBar.start(documents.length, 0);

  // Start the batch embedding process
  await embedder.init();
  await embedder.embedBatch(documents, 1000, async (embeddings) => {
    counter += embeddings.length;
    //Whenever the batch embedding process returns a batch of embeddings, insert them into the index
    await chunkedUpsert(index, embeddings, "default");
    progressBar.update(counter);
  });

  progressBar.stop();
  console.log(`Inserted ${documents.length} documents into index ${indexName}`);
};

run();
```

To run the indexer, use the following command:

```sh
npm run index -- --csvPath=<path-to-csv-file> --column=<column-name>
```

To test our search engine, we'll use the `test.csv` found in the same repo. This file has two columns (`question1` and `question2`) which include similar questions.

To index both columns, we'll run:

```sh
npm run index -- -csvPath=test.csv --column=question1
```

and

```sh
npm run index -- -csvPath=test.csv --column=question2
```

The indexer will set up the index, wait for it to initialize, and then start the embedding process. We should see something like this when then indexer is working:

```sh
Creating index semantic-search
Waiting until index is ready...
Index ready after 45 seconds
Index is ready.
█████████████████████░░░░░░░░░░░░░░░░░░░ 52% | ETA: 291s | 52000/99999
```

## Making queries

Now that our index is populated we can begin making queries. We are performing a semantic search for similar questions, so we should embed and search with another question.

```typescript
import { config } from "dotenv";
import { embedder } from "./embeddings";
import { getPineconeClient } from "./pinecone";
import {
  getEnv,
  getQueryingCommandLineArguments,
  validateEnvironmentVariables,
} from "./utils/util";

config();
const indexName = getEnv("PINECONE_INDEX");

const run = async () => {
  validateEnvironmentVariables();
  const pineconeClient = await getPineconeClient();
  const { query, topK } = getQueryingCommandLineArguments();

  // Insert the embeddings into the index
  const index = pineconeClient.Index(indexName);
  // Initialize the embedder
  await embedder.init();
  // Embed the query
  const queryEmbedding = await embedder.embed(query);

  // Query the index
  const results = await index.query({
    queryRequest: {
      vector: queryEmbedding.values,
      topK,
      includeMetadata: true,
      includeValues: false,
      namespace: "default",
    },
  });

  // Print the results
  console.log(
    results.matches?.map((match) => ({
      // eslint-disable-next-line @typescript-eslint/ban-ts-comment
      // @ts-ignore
      text: match.metadata?.text,
      score: match.score,
    }))
  );
};

run();
```

The querying process is very similar to the indexing process. We create a Pinecone client, select the index we want to query, and then embed the query. We then use the `query` method to search the index for the most similar embeddings. The `query` method returns a list of matches. Each match contains the metadata associated with the embedding, as well as the distance between the query embedding and the embedding in the index.

Let's run some queries and see what we get:

```sh
npm run query -- --query="who is the best Superhero?" --topK=1
```

The result for this will be something like:

```js
[
  {
    id: "1e66157c-6b49-4b53-8148-34b8b5653869",
    score: 0.847043753,
    values: [],
    sparseValues: undefined,
    metadata: { text: "Who yours is the most powerful superhero?" },
  },
];
```
