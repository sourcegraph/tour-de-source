![](https://p21.p4.n0.cdn.getcloudapp.com/items/GGuKDObG/82cb0ac3-427f-4127-835c-eb8c97c7649e.png)

> 💬 **Chat with the creator, Rok Novosel, [in our Discord](https://discord.gg/3NG5E5dH2w)!**

# Code Walkthrough - codesearch.ai

With the rise of transformer neural networks like BERT and GPT-3, we're starting to see more and more applications of machine learning to code. Sourcegraph is experimenting with such models to explore the possibilities to enable better code understanding by humans. As many folks are learning about these models and how to use them, we thought it would be helpful to write up some of our learnings and share an open source example of how to train a model and use it for code search.

[codesearch.ai](https://codesearch.ai) is an experimental AI-powered code search engine. It answers natural language queries with functions indexed from GitHub.com and StackOverflow. Under the hood, it uses the open source Hugging Face Transformers, and the training procedure is inspired by a paper called [Text and Code Embeddings by Contrastive Pretraining](https://arxiv.org/pdf/2201.10005.pdf) from OpenAI. OpenAI also offers an API service for transformer models, but we decided to use Hugging Face for this experiment as the OpenAI models are not open source. The [CodeSearchNet project](https://github.com/github/CodeSearchNet) served as a basis for data collection and cleaning.

In this notebook, we will explore the [codesearch.ai](https://codesearch.ai) implementation, with both high-level descriptions of the approach and embedded code snippets powered by Sourcegraph code intelligence:

1. The first section covers our process for data collection, with example code
2. The second section covers model training using the Hugging Face Transformers API

> This is the third installment of our [Tour de Source](https://tourdesource.substack.com/) series of code-level deep dives into interesting open source projects.

## Data collection

We need a large corpus of natural language queries and corresponding code results to train a semantic code search engine. As you might imagine, there is no readily available dataset we can use, so we have to assemble one on our own. The CodeSearchNet project proposed using open source functions and their docstrings as code and query pairs. Using docstrings as queries is a significant approximation, but it works well in practice. We extend this idea to StackOverflow data by using questions as queries and code from answers. Intuitively, StackOverflow code and query pairs seem better suited for a semantic code search engine, but they are much harder to gather in practice. We're using a StackOverflow data dump available on [archive.org](https://archive.org/download/stackexchange).

### Structure of the Go module

We implemented the data collection procedure in Go. We separated the process into multiple smaller steps, each corresponding to a CLI command. Each command is self-contained and has inspectable output. This makes it easier to manually verify results and shorten the feedback loop while developing. Ideally, we can join individual commands into a single pipeline once confident that it produces consistent results. 

### Initializing the database

We are using a Postgres database as our primary data storage. It will contain processed repositories, extracted functions with metadata, and StackOverflow questions and answers. The web server also uses this database to get additional metadata when returning search results.

To initialize the database run: `go run codesearch-ai-data/cmd/database -init`.

And here is the schema we are using throughout the project:

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cd2b59e71533102da79199031a5270bcc0e0306d/-/blob/codesearch-ai-data/internal/database/database.go?L13-74

### Extracting functions from open source Git repositories

Once we have initialized our database, we can finally start collecting code and query pairs. We will begin by collecting functions and their docstrings from Git repositories. Before starting, we must define the list of repositories we will process. That list should be in the form of a file with one repository per line in the following format: "code-host.url/org/name", for example, `github.com/sourcegraph/sourcegraph`. To start processing, you will have to specify the path to the repository list file and the number of parallel workers.

Example command:

```
go run codesearch-ai-data/cmd/functionextractor -repo-names-file="/path/to/repos.txt" -n-workers=16
```

A simplified procedure for each repository is:

* Check if the repository URL exists
* Check if we have already processed the repository
* Clone the repository in a temporary directory
* Get repository commit ID
* Store the repository in the database
* Walk all files in the repository
  * Only consider files that are smaller than 1MB
  * Avoid minified JavaScript and Protobuf files and files with >1024 columns
  * Get function extractor for file
  * Extract functions from the file
  * Store functions in the database

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cd2b59e71533102da79199031a5270bcc0e0306d/-/blob/codesearch-ai-data/internal/functionextractor/function_extractor.go?L122-208

Let's dive deeper into the function extraction portion. Currently, we support six programming languages: Ruby, Python, PHP, Java, Javascript, and Go. We defined a function extractor for each language using the tree-sitter parser library. Each language has its parsing peculiarities, but function extraction follows the same general pattern:

* Parse the file using tree-sitter
* Walk the AST and identify function and method nodes
* Ignore functions smaller than 4 lines or larger than 512 lines
  * When training, we can only consider a fixed number of tokens so that we can ignore large functions
  * Skipping functions smaller than 4 lines is a heuristic to skip functions with low-information value (getters, setters, etc.)
* Strip comment nodes from the function and reconstruct ("pretty format") the function as a string
* Find the function docstring
  * Usually, we gather the comments preceding the function node
  * Python is a special case because the docstring lives as the first string node inside the function
  * Remove comment delimiters from comment strings (#, /*, *, */, //)
* Store all **unique** extracted functions in the database
  * Extracted functions consist of: code, code hash, docstring, inline comments, function identifier, start line, end line
  * Duplicated functions negatively affect the model training performance, so we have to deduplicate them at this stage
  * We concatenate docstrings and inline comments from duplicated functions

We can observe the above pattern in the Go function extractor.

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cd2b59e71533102da79199031a5270bcc0e0306d/-/blob/codesearch-ai-data/internal/functionextractor/go_function_extractor.go?L30-54

You might be wondering why we decided to strip comments from the function code? The working hypothesis is that it would force the model to strictly focus on the code when trying to learn the vector representation and not mix it with natural language from the comments. Beyond that, it would also reduce the function size so we can fit more code tokens when training the model. Below we have the `StripComments` function, which takes a root node and returns a list of non-comment nodes and a list of comment nodes.

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cd2b59e71533102da79199031a5270bcc0e0306d/-/blob/codesearch-ai-data/internal/parsinghelpers/strip_comments.go?L11-48

We take the list of comment nodes, remove the comment delimiters and join them into a string. The list of non-comment nodes has to be reconstructed into a string while maintaining its original structure. We call this "pretty formatting," implemented in the `PrettyFormatNodes` function below.

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cd2b59e71533102da79199031a5270bcc0e0306d/-/blob/codesearch-ai-data/internal/parsinghelpers/pretty_format.go?L23-56

Pretty formatting the non-comment nodes also has an additional benefit: introducing a semi-uniform structure to all functions across languages. It enforces spaces instead of tabs and reduces the space between tokens to one. Here is an example of a function before and after comment stripping and pretty formatting:

Before:
```go
package abc

/*
Comment 1
*/
// Comment 2
func a() int {
    return 1 + 1
}

// Comment 3

// Comment 4
// Comment 5
func b() {
    // Comment 6

    // Comment 7
    c := func() string {
        return "a"
    }
}
```

After:
```go
package abc
func a() int {
 return 1 + 1
}
func b() {
 c := func() string {
  return "a"
 }
}
```

### StackOverflow data

The following data source we are using is the StackOverflow questions and answers collection hosted by [archive.org](https://archive.org/download/stackexchange). Specifically, we are using the `stackoverflow.com-Posts.7z` file. It contains a single `Posts.xml` with the following structure:

```html
<?xml version="1.0" encoding="utf-8"?>
<posts>
  <row Id="4" PostTypeId="1" AcceptedAnswerId="7" CreationDate="2008-07-31T21:42:52.667" Score="756" ViewCount="63468" Body="&lt;p&gt;I want to use a &lt;code&gt;Track-Bar&lt;/code&gt; to change a &lt;code&gt;Form&lt;/code&gt;'s opacity.&lt;/p&gt;&#xA;&lt;p&gt;This is my code:&lt;/p&gt;&#xA;&lt;pre class=&quot;lang-cs prettyprint-override&quot;&gt;&lt;code&gt;decimal trans = trackBar1.Value / 5000;&#xA;this.Opacity = trans;&#xA;&lt;/code&gt;&lt;/pre&gt;&#xA;&lt;p&gt;When I build the application, it gives the following error:&lt;/p&gt;&#xA;&lt;blockquote&gt;&#xA;&lt;pre class=&quot;lang-none prettyprint-override&quot;&gt;&lt;code&gt;Cannot implicitly convert type decimal to double&#xA;&lt;/code&gt;&lt;/pre&gt;&#xA;&lt;/blockquote&gt;&#xA;&lt;p&gt;I have tried using &lt;code&gt;trans&lt;/code&gt; and &lt;code&gt;double&lt;/code&gt;, but then the &lt;code&gt;Control&lt;/code&gt; doesn't work. This code worked fine in a past VB.NET project.&lt;/p&gt;&#xA;" OwnerUserId="8" LastEditorUserId="3072350" LastEditorDisplayName="Rich B" LastEditDate="2021-02-26T03:31:15.027" LastActivityDate="2021-11-15T21:15:29.713" Title="How to convert a Decimal to a Double in C#?" Tags="&lt;c#&gt;&lt;floating-point&gt;&lt;type-conversion&gt;&lt;double&gt;&lt;decimal&gt;" AnswerCount="12" CommentCount="4" FavoriteCount="59" CommunityOwnedDate="2012-10-31T16:42:47.213" ContentLicense="CC BY-SA 4.0" />
  <row Id="6" PostTypeId="1" AcceptedAnswerId="31" CreationDate="2008-07-31T22:08:08.620" Score="313" ViewCount="22477" Body="&lt;p&gt;I have an absolutely positioned &lt;code&gt;div&lt;/code&gt; containing several children, one of which is a relatively positioned &lt;code&gt;div&lt;/code&gt;. When I use a &lt;code&gt;percentage-based width&lt;/code&gt; on the child &lt;code&gt;div&lt;/code&gt;, it collapses to &lt;code&gt;0 width&lt;/code&gt; on IE7, but not on Firefox or Safari.&lt;/p&gt;&#xA;&lt;p&gt;If I use &lt;code&gt;pixel width&lt;/code&gt;, it works. If the parent is relatively positioned, the percentage width on the child works.&lt;/p&gt;&#xA;&lt;ol&gt;&#xA;&lt;li&gt;Is there something I'm missing here?&lt;/li&gt;&#xA;&lt;li&gt;Is there an easy fix for this besides the &lt;code&gt;pixel-based width&lt;/code&gt; on the child?&lt;/li&gt;&#xA;&lt;li&gt;Is there an area of the CSS specification that covers this?&lt;/li&gt;&#xA;&lt;/ol&gt;&#xA;" OwnerUserId="9" LastEditorUserId="9134576" LastEditorDisplayName="user14723686" LastEditDate="2021-01-29T18:46:45.963" LastActivityDate="2021-01-29T18:46:45.963" Title="Why did the width collapse in the percentage width child element in an absolutely positioned parent on Internet Explorer 7?" Tags="&lt;html&gt;&lt;css&gt;&lt;internet-explorer-7&gt;" AnswerCount="7" CommentCount="0" FavoriteCount="13" ContentLicense="CC BY-SA 4.0" />
  <row Id="7" PostTypeId="2" ParentId="4" CreationDate="2008-07-31T22:17:57.883" Score="501" Body="&lt;p&gt;An explicit cast to &lt;code&gt;double&lt;/code&gt; like this isn't necessary:&lt;/p&gt;&#xA;&#xA;&lt;pre&gt;&lt;code&gt;double trans = (double) trackBar1.Value / 5000.0;&#xA;&lt;/code&gt;&lt;/pre&gt;&#xA;&#xA;&lt;p&gt;Identifying the constant as &lt;code&gt;5000.0&lt;/code&gt; (or as &lt;code&gt;5000d&lt;/code&gt;) is sufficient:&lt;/p&gt;&#xA;&#xA;&lt;pre&gt;&lt;code&gt;double trans = trackBar1.Value / 5000.0;&#xA;double trans = trackBar1.Value / 5000d;&#xA;&lt;/code&gt;&lt;/pre&gt;&#xA;" OwnerUserId="9" LastEditorUserId="5496973" LastEditDate="2019-10-21T14:03:54.607" LastActivityDate="2019-10-21T14:03:54.607" CommentCount="0" ContentLicense="CC BY-SA 4.0" />
</posts>
```

The extracted `Posts.xml` file takes up around ~90GB of disk space, so reading the entire file into memory is not manageable. Thankfully, they structured the file so that each row is on its own line, and we can process the file in a streaming fashion.

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cd2b59e71533102da79199031a5270bcc0e0306d/-/blob/codesearch-ai-data/internal/soimporter/import.go?L62-118

We do not store all of the available questions and answers. Instead, we try to filter out as many questions and answers as possible to save on disk space and processing time. We ignore questions and answers with a score less than 0 and all questions without answers. Question and answer bodies are represented as encoded HTML strings. We decode them before storing them in the database. At this point, we are not extracting any code from the questions or answers.

### Inspecting extracted functions and questions

We extracted millions of functions and millions of StackOverflow questions with corresponding answers. Now we need a way to inspect the vast quantity of gathered entities. Manual verification is time-consuming, but confirming our data pipeline is working well is necessary. We also have an array of unit tests to ensure we're not introducing any regressions.

You can run the inspect web server using the command `go run codesearch-ai-data/cmd/inspect`. Extracted functions are available on `localhost:8080/inspect/extracted-functions`, and StackOverflow questions are available on `localhost:8080/inspect/so-questions`.

### Marking train repositories

In machine learning, when doing the train/test split, we usually assume that samples are independent. We cannot make the same assumption for extracted functions in our case. Functions from the same repository generally have the same structure, naming pattern, variable usage, etc. If we uniformly split the extracted functions into train and test sets, we would inadvertently leak that information and possibly get artificially better evaluation scores. To avoid this, we instead split repositories into train and test repositories. We will use functions from train repositories to train the model and functions from the test repositories for evaluation.

To mark train repositories, run the command 

```
go run codesearch-ai-data/cmd/marktrainrepos -train-test-ratio=0.99
```

and specify the train/test split ratio.

### Code and query pairs

We are finally ready to convert our collection of functions and StackOverflow data into code and query pairs ready for model training.

To import code and query pairs, run: 

```
go run codesearch-ai-data/cmd/codequerypairsimporter -so -extracted-functions -so-train-test-ratio=0.99
```

Extracted functions already nicely map to the code and query pairs concept. We have already cleaned up the function code and gathered the associated docstrings to use as queries. The only remaining thing to handle is functions without docstrings. We saved function names and inline comments precisely for this use case. We split the function identifier into parts (e.g., MyFunctionName into My function name) and append the inline comments to form a docstring. We can then use the constructed docstring as a query. Finally, we filter out any queries with less than four words since they are most likely not informative enough. We also remove all non-ASCII characters from query strings.

Here is the code that constructs a code and query pair from an extracted function:

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cd2b59e71533102da79199031a5270bcc0e0306d/-/blob/codesearch-ai-data/internal/codequerypairsimporter/extractedfunctions.go?L66-83

The method for converting StackOverflow questions and answers to code and query pairs is slightly more involved. We start by joining answers against the questions table, grouping the results by questions, and aggregating the answers into an array. The query is simply the question title with non-ASCII characters removed. We construct the code part from the code found in the answers.

Extracting the raw code from the answers was a lot trickier than it initially seemed. Since the code inside was not properly escaped, we could not use a library like `goquery` to extract all `<code>` elements. The biggest issue was PHP code that started with `<?php` since `goquery` parsed it as a valid HTML element, thus mangling the raw text content. To account for this edge case, we rolled out a custom solution to manually find the code snippets:

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cc20497d1b3614fcad36e658543211fce0b4c886/-/blob/codesearch-ai-data/internal/socode/so_code_extractor.go?L19-72

We're not parsing the HTML per se, mostly just finding the raw string between two most outer `<code>` and `</code>` tags.

Once we have the raw code snippets from the answers, it is time to give the same treatment as the extracted functions. We use the question tags to find the most likely parsers to parse the answers (posters usually include programming language in the tags). Before parsing, we remove any dots (`...`) that indicate skipped code, which could break otherwise parseable code. Next, we parse the code snippet, remove the inline comments, and format it back into a string. PHP is again an issue since many of the code snippets are missing the `<?php`  tag, so we have to append it. In the end, we deduplicate the code found in the answers and only keep the code longer or equal to 10 characters.

As for the train/test split for questions, we have a more straightforward solution. We supply a train/test split ratio to the command and evaluate `rand.Float64() < trainTestSplitRatio` for each question to decide whether it is in the training set.

The code is slightly convoluted, but here is the described functionality:

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cc20497d1b3614fcad36e658543211fce0b4c886/-/blob/codesearch-ai-data/internal/codequerypairsimporter/so.go?L123-206

### Output training data

We are finally at the end of the data collection phase. We are ready to convert our meticulously gathered code and query pairs into something our training phase can use. Theoretically, the training phase could read the pairs directly from the Postgres database, but we could not find a way to integrate them with the Hugging Face datasets library. Instead, we decided to output all of the code and query pairs into JSONL (JSON lines) files, where each line contains a JSON-encoded code and query pair with associated metadata.

In total, we output eight files:
* train.jsonl, test.jsonl: Used for model pretraining
* so.jsonl, so.train.jsonl, so.test.jsonl: Used to fine-tune the code search task on StackOverflow data
* extracted-functions.jsonl, extracted-functions.train.jsonl, extracted-functions.test.jsonl: Used to fine-tune the code search task on functions

We won't need all the files, but they are helpful if you only want to experiment with a specific subset of data.

To output the training data, run: 

```
go run codesearch-ai-data/cmd/outputtrainingdata -output-directory=/path/to/output -train -test -so -extracted-functions
```

## Training the code search model

The idea behind using neural networks for code search is to represent code and text (i.e., docstrings) as vectors in the same high-dimensional vector space. That allows us to use distance measures (like cosine) and the nearest neighbor search algorithm to build an index of code vectors produced by the neural network. We can then search the index using the query (docstring) vector made by the same network to find code snippets that are semantically close.

We could take multiple approaches to train a code search neural network. For example, the CodeSearchNet project uses an encoder for the queries and one encoder for each programming language it supports. An encoder is responsible for converting text into a vector representation. CodeSearchNet supports a multitude of encoders like a neural bag of words, RNN, 1D-CNN, etc. The training procedure aims to minimize the cosine distance between the corresponding query and code vector pairs and maximize the distance between non-corresponding pairs. The goal is that each encoder arrives at the same vector space representation that we can use for effective code search.

Then came the era of Transformers. OpenAI introduced a paper called [Text and Code Embeddings by Contrastive Pretraining](https://arxiv.org/pdf/2201.10005.pdf) that simplified the training procedure and massively improved the overall performance scores. They propose using a single transformer encoder to encode query and code strings instead of separate encoders for each possible input. The intuition behind a single encoder is that it will learn the embeddings better since we train query and code representations using the same weights. The difference between query and code inputs is in the special delimiter tokens. When tokenizing query and code strings, they use separate SOS (start of sentence) and EOS (end of sentence) tokens for query and code inputs.

Ok, enough intro. The following sections will explain how we train the code search model without going too deep into the machine learning weeds. If you want to learn more about Transformers, I suggest reading the excellent [Hugging Face docs](https://huggingface.co/docs/transformers/index).

### Training the tokenizer
We need a way of transforming raw input strings into vectors of numbers. That is the responsibility of the tokenizer. It takes a string, breaks it into pieces, and converts them into tokens that our model can use. Since we're training our model from scratch, we cannot use a predefined tokenizer, and we'll also have to prepare it from scratch. We are using the `ByteLevelBPETokenizer` class. We train the tokenizer by feeding it all the code and query strings in the `train.jsonl` file. It builds a vocabulary of all the subword tokens and merges it will use to tokenize the string.

The function that implements the training:

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cc20497d1b3614fcad36e658543211fce0b4c886/-/blob/codesearch_ai_ml/train_tokenizer.py?L62-88

The command to train the tokenizer is: 

```
python codesearch_ai_ml/train_tokenizer.py --code-query-pairs-file=train.jsonl --output-dir=output
```

### Preparing the datasets
Using Hugging Face Datasets is not strictly necessary, but it does speed up the operations down the line. We prepare three datasets: tokenized dataset, pretraining dataset, and fine-tuning dataset. The tokenized dataset contains the tokenized code and query strings, and it discards unnecessary columns. The pretraining dataset uses the tokenized dataset to split the code and query columns and join them into a new dataset. The fine-tuning dataset also uses the tokenized dataset, but it simply filters out any rows that have an empty query.

The commands to prepare the mentioned datasets:

Tokenized dataset: 
```
python codesearch_ai_ml/prepare_tokenized_datasets.py --code-query-pairs-file=train.jsonl --model-path=tokenizer --output-dir=output
```
Pretraining dataset:

```
python codesearch_ai_ml/prepare_pretraining_datasets.py --dataset=dataset --output-dir=output
```

Fine-tuning dataset: 

```
python codesearch_ai_ml/prepare_fine_tuning_datasets.py ---dataset=dataset --output-dir=output
```

### Pretraining the transformer model
Finally, we are ready to train some models. Specifically, we can start pretraining the Hugging Face RoBERTa model using masked-language modeling. Here is the model config:

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cc20497d1b3614fcad36e658543211fce0b4c886/-/blob/codesearch_ai_ml/model_config.py?L2-5

Hugging Face makes it easy to cut down on the boilerplate required to train a custom model:

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cc20497d1b3614fcad36e658543211fce0b4c886/-/blob/codesearch_ai_ml/pretrain_language_model.py?L21-73

To pretrain the model, run: 

```
torchrun --nproc_per_node=$N_GPUS codesearch_ai_ml/pretrain_language_model.py --model-path=/path/to/model --train-dataset=/path/to/train --test-dataset=/path/to/test
```

We are using `torchrun` to distribute training across all available GPUs.

### Fine-tuning the model for code search
Once we have pretrained the base model, we can fine-tune it on the code search task. Hugging Face does (yet) not have a code search-specific model, so we rolled out our own using PyTorch:

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cc20497d1b3614fcad36e658543211fce0b4c886/-/blob/codesearch_ai_ml/fine_tune_model.py?L24-48

The model is taken from the OpenAI paper with slight modifications due to the RoBERTa encoder model we are using. The final significant bit is calculating the loss:

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cc20497d1b3614fcad36e658543211fce0b4c886/-/blob/codesearch_ai_ml/fine_tune_model.py?L85-97

To start fine-tuning the model, run: 

```
python codesearch_ai_ml/fine_tune_model.py --base-model=models/base --fine-tuned-model=models/fine-tuned --train-dataset=fine-tuning/train --test-dataset=fine-tuning/test
```

### Training the FAISS index

We're using the FAISS library to build the nearest neighbor search index. Using the fine-tuned model, we encode all the code snippets, train the FAISS index on a subset of encoded snippets, and add all snippets to the index.

FAISS index config:

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cc20497d1b3614fcad36e658543211fce0b4c886/-/blob/codesearch_ai_ml/prepare_faiss_index.py?L23-30

To train the index, run: 

```
python codesearch_ai_ml/prepare_faiss_index.py --base-model=models/base --fine-tuned-model=models/fine-tuned --code-query-pairs-file=all-pairs.jsonl --output-dir=faiss --n-gpus=$N_GPUS
```

Here is an example of how to search the index:

https://sourcegraph.com/github.com/sourcegraph/codesearch.ai@cc20497d1b3614fcad36e658543211fce0b4c886/-/blob/codesearch_ai_ml/search.py?L14-25

## Further exploration

[Codesearch.ai](https://codesearch.ai/) is far from perfect and there's still much work to be done in the realm of applying machine learning to human code understanding. As we continue to explore this area, we'd love to hear from you if you have feedback as a user or would like to collaborate with us. Pop into [our Discord](https://discord.gg/SSCBGByJeu) to say hi!

If you'd like to support our efforts as a business, you can spin up a Sourcegraph instance--we build a code search engine that alleviates the pain of working in large codebases. You can get it up and running in 5 minutes with a [single `docker run` command](https://docs.sourcegraph.com/#getting-started). If you try it out, feel free to pop into the aforementioned Discord and pepper us with any questions!

## Next Steps

* 💬 **Chat with the creator, Rok Novosel, [in our Discord](https://discord.gg/3NG5E5dH2w)!**
* 🔍 [Browse the code](https://sourcegraph.com/github.com/sourcegraph/codesearch.ai)
* 📧 [Subscribe to Tour de Source](https://tourdesource.substack.com/)
