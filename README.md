# Conversational Search
Truly Conversational Search is the next logic step in the journey to generate intelligent and useful AI. To understand what this may mean, researchers have voiced a continuous desire to study how people currently converse with search engines. 


## Introduction
Traditionally, the desire to produce such a comprehensive dataset has been limited because those who have this data (Search Engines) have a responsibility to their users to maintain their privacy and cannot share the data publicly in a way that upholds the trusts users have in the Search Engines. Given these two powerful forces we believe we have a dataset and paradigm that meets both sets of needs: A artificial public dataset that approximates the true data and an ability to evaluate model performance on the real user behavior. What this means is we released a public dataset which is generated by creating artificial sessions using embedding similarity and will test on the original data. To say this again: we are not releasing any private user data but are releasing what we believe to be a good representation of true user interactions.

## Corpus Generation
To generate our projection corpus, we took the 1,010,916 MSMARCO queries and generated the query vectors for each unique queries. Once we had these embedding spaces, we build an Approximate Nearest Neighbor Index using [ANNOY](https://github.com/spotify/annoy).
Next, we sampled our Bing usage log from 2018-06-01 to 2018-11-30 to find a sample of sessions that that had more than 1 query, shared a query that had a query embedding similar to a MSMARCO query, and were likely to be conversational in nature. Next we remove all navigation, bot, junk, and adult sessions. Once we did this, we now had 45,040,730 unique user sessions of 344,147 unique queries. The average session was 2.6 queries long and the longest session was 160 queries. Just like we did for our public queries, we generated embedding for each unique query. Finally, in order to merge the two, for each unique session we perform a nearest neighbor search given the real queries query vector in the MSMARCO ANN Index. This allows us to join the public queries to the private sessions generating an artificial user session grounded in true user behavior. 

An example of these search sessions is below.
```
marco-gen-dev-40        what is the australian flag     what is the population of australia     what hemisphere is north australia                                                                       how big is sydney australia      is australia a country
marco-gen-dev-152       is elements on a periodic table what is the product of ch4      cost of solar system    what constellation is cassiopeia in                                                      what is the human skeleton       what is a isosceles triangle    convert a fraction to a % calculator    standard deviation difference calculator                                                         graph y = 1/2 x cubed how to     what is the volume of a pyramid what is american sign language definition       how to put word count on word
marco-gen-dev-157       define colonialism      what are migrant workers        office 365 cost to add a user   what does per capita means                                                               what are tariffs definition for tariffs  define urbanization
marco-gen-dev-218       icd 10 code rhinitis    icd diagnosis code for cva      icd code for psoriatic arthritis       icd 10 code for facet arthritis of knee                                           icd 10 code for copd     icd codes for cad       icd diagnosis code for cva      icd 10 code for personal history pvd
marco-gen-dev-385       circadian activity rhythms definition   what is the synonym of insomnia insomnia definition    define narcolepsy                                                                 causes for night terrors types of meditation     definition: meditation  pros and cons for death penalty
marco-gen-dev-397       cost of solar system    what is the hottest planet?     what is coldest planet  what is solar system is                                                                          what kind of galaxy is the milky way     spanish iris is what in english
marco-gen-dev-457       can we repeal donald trump      what was barack obama   is john mccain a republican     who is chuck schumer's daughter                                                          was bill clinton a democrat      is mike pence a veteran why did barack obama get a nobel peace prize    was mahatma gandhi awarded a peace prize                                                         why did dalai lama win the nobel peace prize
marco-gen-dev-485       definition of technology        define scientific method        definition of constant  graphs definition                                                                        definition of information technology     define axis     definition of  trial    legal definition of bias        define what an inference is
marco-gen-dev-496       illusory definition     illusion vs allusion definition declaration + definition        define: caste                                                                            define rhetoric  define race
marco-gen-dev-572       stock price tesla       fb stock price  home depot stock price t        amazon stock price     nxp semiconductors stock price
```

We first release [BERT Based Sessions](https://msmarco.blob.core.windows.net/conversationalsearch/artificialSessionsBERT500k.tsv.gz) and [Query Embedding Based Sessions](https://msmarco.blob.core.windows.net/conversationalsearch/artificialSessionsQueryEncoding500kSample.tsv.gz) which we shared with a small group of researchers to get feedback on what worked better. Based on community feedback and data exploration our we are releasing our first full scale dataset described below.

1. Split the 45,040,730 artificial sessions into a train, dev and test. To do so, any session that included a query from the QnA eval set was considered eval and the remaining sessions were split 90%/10% between train and dev. These files are called full_marco_sessions_ann_split.* and have the following sizes
```
spacemanidol@spacemanidol:/mnt/c/Users/dacamp/Desktop$ wc -l full_marco_sessions_ann_split.*
   3656706 full_marco_sessions_ann_split.dev.tsv
   8480280 full_marco_sessions_ann_split.test.tsv
  32903744 full_marco_sessions_ann_split.train.tsv
  45040730 total
```
2. Use the query vectors to generate a similairty score between edges and filter out four types of edges
    * Topic Change: (cosine similarity) <= 0.4 
    * Explore: (cosine similarity) \in (0.4, 0.7]
    * Specify: (cosine similarity) \in (0.7, 0.85]
    * Paraphrase: (cosine similarity) \in (0.85, 1]
3. Filter by session coherence
    * treat "topic change" as non-related and no edge, 
    * build the graph in each session, 
    * pick the biggest sub-graph in the session, 
    * throw away the rest queries not in the biggest sub-graph
4. Filter by session length removing anything with less than 4 queries.
5. Filter any session where the chain of queries were just paraphrases

Upon doing this, we have 3 sets of files: [Train](https://msmarco.blob.core.windows.net/conversationalsearch/ann_session_train.tar.gz),[Dev](https://msmarco.blob.core.windows.net/conversationalsearch/ann_session_dev.tar.gz), and eval. We are not currently sharing eval. Explicit size numbers below
```
    75193 marco_ann_session.dev.all.tsv
   774724 marco_ann_session.test.all.tsv
   675334 marco_ann_session.train.all.tsv
  1525251 total
```

To provide more exploratory content, we further filter the sessions in each split(Train, Dev, Eval) using three types of filtering:
1. ".half_trans.jsonl": half (explore|specifiy) >= 50% adjacent edges in (0.4, 0.85]
2. ".half_explore.jsonl": half (explore): >= 50% adjacent edges in (0.4, 0.7]
3. ".half_specify.jsonl": half (specifiy): >= 50% adjacent edges in (0.7, 0.85]

Explicit size numbers below
```
 spacemanidol@spacemanidol:/mnt/c/Users/dacamp/Desktop$ wc -l marco_ann_session.*.half*
    53791 marco_ann_session.dev.half_explore.tsv
     5646 marco_ann_session.dev.half_specify.tsv
    63854 marco_ann_session.dev.half_trans.tsv
   548101 marco_ann_session.test.half_explore.tsv
    42032 marco_ann_session.test.half_specify.tsv
   648868 marco_ann_session.test.half_trans.tsv
   482021 marco_ann_session.train.half_explore.tsv
    50980 marco_ann_session.train.half_specify.tsv
   572791 marco_ann_session.train.half_trans.tsv
  2468084 total
```


## Corpus task
We are currently assembling baselines and building an evaluation framework but the initial task will be as follows: Given a chain of queries q1, q2,...,q(n-1) predict q(n). Ideally systems will train and test on the public artificial and will be evaluated on both the private real data and public artificial eval data.

### Initial 500k Sample Process
1. Downloaded the data
~~~
./downloadData.sh
~~~
2. Generate the querySets
~~~
python generateQuerySets.py <msmarco train queries> <msmarco dev queries> <msmarco eval queries> <quoraQueries> <NQFolder>
~~~
3. Get Query Embeddings
~~~
python generateQueryEmbeddings.py <url> allQueries.tsv queryEmbeddings.tsv
~~~
4. Generate BERT Query Embeddings
To generate our Query Embeddings [we used BERT As A Service](https://github.com/hanxiao/bert-as-service) to generate a unique query embedding for each query in our set. If you want to go ahead and regenerate embeddings(or use it to generate another alternate query source for you model) you can follow what we did below.
~~~
cd Data/BERT
pip install bert-serving-server  # server
pip install bert-serving-client  # client, independent of `bert-serving-server`
wget https://storage.googleapis.com/bert_models/2018_10_18/cased_L-24_H-1024_A-16.zip
unzip cased_L-24_H-1024_A-16
bert-serving-start -model_dir ~/Data/BERT/cased_L-24_H-1024_A-16  -num_worker=4 -device_map 0 #depending on your computer play around with these settings
~~~
In a separate shell
~~~
python3 generateQueryEmbeddingsBERT.py allQueries.tsv BERTQueryEmbeddings.tsv
~~~
5. Generate Sessions
~~~
python generateArtificialSessions.py realQueries queryEmbeddings.tsv BERTQueryEmbeddings.tsv queryEmbeddings.ann sessions.tsv
~~~


# Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
