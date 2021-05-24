---
layout: post
title: "Machinehack - Book price prediction"
---


# Book price prediction

Current standing: #36

## EDA & Feature Engineering

This is a very challenging problem for a beginner like me because it combines multiple techniques together to get a good result. These include:

- Text processing. For columns Title, Synopsis, Genre
- Categorical processing. For columns: Edition, BookCategory,
- Feature engineering:
    - Use Edition to infer columns: Year, Month, Day, IsImported, PrintEdition,
    - Use Genre to infer columns: IsBook
    - Use Author to infer columns:
        - IsMultipleAuthor: whether a title is written by multiple authors. This is the case when there are more than 1 author names listed separated by ','
        - IsSpecialAuthor: whether a title is written by someone with a personal title like Dr., PhD., Prof. and so on
        - IsFamousAuthor: whether an author has more than 100 reviews
- Missing value handling:
    - Some rows miss the value Year. We just need to randomize from min year and max year
    - Some rows miss the value Month. We just need to randomize from the list of available months
    - Some rows miss the value Day. We need to randomize the day available to respective value from Month column
- Invalid text handling:
    - The column Edition contains value `-` with is similar to the minus sign but it's not.

## Technical learnings

### General things

Here are the things I learned from this competition: 

- String column can be queried using `column.str.<function>`
- Pandas dataframe can be queried and assigned value to using `df.loc[query, column(s)]`. This example show how the year column was set by using querying & apply

```python
df.loc[pd.isnull(df.Year), 'Year'] = df.loc[pd.isnull(df.Year), 'Year'].apply(lambda x: random.randrange(df.Year.min(), df.Year.max()))
```

- `.apply` function is really powerful. This example show how the `IsMultipleAuthor` was set using apply.

```python
df['IsMultipleAuthor'] = df.Author.str.match('.*[,&-].*')
```

- Keras is a good tool for quickly building up an MLP model, as an alternative to `from sklearn.neural_network import MLPRegressor`. It has several niceties such as:
    - Keras wrapper that works well with the rest of sklearn ecosystem
    - Early stopping.
    - There's a nice package name `PlotLossesKeras` to plot live graph of train/valid loss during training.
- A nice trick that I learned is that we should merge the train/submission data and process together. It saves us the time not to come back and redo all the preprocessing steps for a different data frame. This is especially painful when using jupyter notebook

```python
train_df['type'] = 'train'
test_df['type'] = 'test'
df = pd.concat([train_df, test_df], axis=0)
# start doing preprocessing
train_df = df[df.type=='train']
test_df = df[df.type=='test']
```

- People usually have a list of libraries they use very often. I will define one I use a lot and in my next challenge I will paste the whole list of import instead of typing each one in succession.
- Very cool trick of applying frequency mapping to categorical column. What this effective does is:
    - Count the frequency of each value in the BookCategory column
    - Assign the value of `category` to the frequency of the BookCategory's value. For example, if the BookCategory `Action & Adventure` appears 1036 times, then the value for `category` for that row will be 1036. Of course this value will be treated as numeric data later on.

```python
df['category'] = df.BookCategory.map(df.BookCategory.value_counts())
```

### sklearn pipelines

sklearn pipeline is a tricky concept to understand. I will not go too much into details how to use it. What I learned about it is that it helps reduce repetitive tasks by using reusable and predefine templates. 

- If you have categorical columns, most likely you will want to turn in into one-hot encoding. To do that
- If you have int columns, usually you would want to apply MinMaxScaler to it
- If you have boolean column, you will want to turn it into an int column for regression for this challenge
- Text data is quite tricky to put into a pipeline in sklearn. CountVectorizer, TfidfVectorizer and TfidfTransformer don't work well with text column. In this challenge, I needed to use FunctionTransformer with my custom function to do all-in-one processing from text to tfidf feature vector. Things to note:
    - I used very simple tokenization and then lemmatization on the text. After that I joined them back to a string so that the next item in the pipeline (which is the count vectorizer) can work.
    - The results matrix of tfidf transformer would be a very large & sparse matrix. The result of the whole pipeline for the whole initial data frame would be a very large & sparse matrix, since the result matrix from tfidf is joined with other matrixes resulted from processing other columns. I didn't realize this until later I fit the LinearModel to the result matrix - it didn't take much time before but after putting the text processing into a pipeline, it took a lot more time. One trick I needed to employ was to convert the result matrix to a scipy sparse matrix. It's actually the result type of pandas pipeline anyway, so the rest of the code works fine.

```python
stop = stopwords.words("english")

categorical_features = ["BookCategory", "Month", "PrintEdition", "Day"]
int_features = ["Year"]
bool_features = [
    "IsBook",
    "IsImported",
    "IsMultipleAuthor",
    "IsSpecialAuthor",
    "IsFamousAuthor",
]
text_features = ["Synopsis", "Title", "Author"]

def lemmatize(s):
    dff = []
    for n, c in s.iteritems():
        dff.append(
            c.str.lower().apply(
                lambda t: " ".join(
                    [Word(i).lemmatize() for i in t.split() if i not in stop]
                )
            )
        )
    dff = pd.concat(dff, axis=1)
    return dff

def tovector(s):
    dff = []
    for n, c in s.iteritems():
        dff.append(
            c.apply(
                lambda x: np.mean(
                    [(word_model[i] if i in word_model else np.zeros(100)) for i in x],
                    axis=0,
                )
                if x
                else np.zeros(100)
            )
        )
    dff = pd.concat(dff, axis=1)
    return dff

def text_vector_to_column(s):
    dff = []
    for n, c in s.iteritems():
        for i in range(100):
            dff.append(c.apply(lambda x: x[i]))
    dff = pd.concat(dff, axis=1)
    return dff

def as_int(s):
    dff = pd.concat([c.astype(int) for n, c in s.iteritems()], axis=1)
    return dff

def text_pro(s):
    dff = []
    for n, c in s.iteritems():
        count_vectorizer = CountVectorizer()
        tfidf_vectorizer = TfidfVectorizer()
        count = count_vectorizer.fit_transform(c)
        dff.append(pd.DataFrame.sparse.from_spmatrix(tfidf_vectorizer.fit_transform(c)))
    return pd.concat(dff, axis=1)

text_transformer = Pipeline(
    [
        ("tokenize_lemmatize", FunctionTransformer(lemmatize)),
        ("count_vectorizer", FunctionTransformer(text_pro)),
    ]
)

bool_transformer = Pipeline([("to_int", FunctionTransformer(as_int))])

categorical_transformer = Pipeline(
    [
        ("onehot", OneHotEncoder())
    ]
)

int_transformer = Pipeline([("minmaxscaler", MinMaxScaler())])

preprocessor = ColumnTransformer(
    transformers=[
        ("cat", categorical_transformer, categorical_features),
        ("text", text_transformer, text_features),
        ("bool", bool_transformer, bool_features),
        ("int", int_transformer, int_features),
    ],
    verbose=True,
)

def metric_np(y_pred, y_true):
    y_pred = math.e ** y_pred
    y_true = math.e ** y_true
    return 1 - np.sqrt(np.square(np.log10(y_pred + 1) - np.log10(y_true + 1)).mean())

# train_df.head()
y = np.array(train_df.Price).reshape(-1, 1)
x = preprocessor.fit_transform(df)
x = scipy.sparse.csr_matrix(x)
x.shape
```

A few things I learned about pipeline:

- One can turn any function into a transformer by using `FunctionTransformer`
    - The function will receive a data frame. It needs to return a data frame. This is generally not strictly required, but for a uniform usage of sequential transformers, I think this should be generally encouraged.
- Pipeline is quite hard to grasp, but I realized this is a very good method to use. It helps modularize and reuse a lot of code. Prior to using pipeline after feature engineering I needed to transform the columns manually before fitting into a model. Any changes made to the feature I will have to make change to this transformation code as well. However with pipeline, this is no longer the case - the transformation code mostly stays the same.

## Cross validation

I understand more about cross validation. One question I had was: why was cross validation needed when it doesn't produce any usable model for inferring later. The answer I learned is: cross validation is not used to pick the best model, it's used for picking the best kind of model. 

Without cross validation, a model's performance would be affected by the choice of train/test split. If train split is full of easy samples and test split is full of hard sample, the performance is gonna be really bad. Conversely, if train split is full of hard samples and test split full of easy samples, the test loss is going to be smaller than that in the train split. In both cases, the prediction quality in unseen data is not very predictable. 

Cross validation does away with this concern. By splitting the data into multiple folds  and then training/testing on different folds, one can stop worrying about the issue above. 

It's also provide a basis to choose from different kinds of model. For example, how would one choose using `LinearRegression` over `MLP`? With cross validation, we can see that range of performance each model makes and choose one with better average performance. In this particular problem, MLP performs way better than LinearRegression on average (or rather, most of the cases)

GridSearchCV (grid search cross validation) simply bring this idea to the next level. It uses the  cross validation performance during grid searching to make sure that we can find the best params to provide the best performance on average.