## Overview

- The goal of this demo is to use elasticsearch to query data and retrieve relevant information. This is part of our elasticsearch learning journey.

- This demo make use of kaggle [Amazone product sales dataset 2023](https://www.kaggle.com/datasets/lokeshparab/amazon-products-dataset/data?select=Amazon-Products.csv).



keyword: Elasticsearch, kibana, query data, kaggle Amazone product sales dataset

> [!NOTE]
> The dataset contains many csv file but only`Amazon-Products.csv` will be used since it combine others file chunks already.


## Step 1: Data cleaning

This step goal is to generate `amazon_products_cleaned.csv` in `cleaned_dataset` folder and will be used further for the elasticsearch queries.

1. Download, extract the dataset from [kaggle](https://www.kaggle.com/datasets/lokeshparab/amazon-products-dataset/data?select=Amazon-Products.csv). 
2. Put `Amazon-Products.csv` in `amazon_product_dataset` folder.
3. Use `clean_dataset.ipynb` notebook to clean the dataset and generate `amazon_products_cleaned.csv` file located in `cleaned_dataset` folder. 


## Step 2: start elasticsearch and kibana in docker

```
cd elasticsearch_kibana_docker
```

Launch docker desktop run the following command to start elasticsearch and kibana containers in docker

```
docker compose up -d
```

## Step 3: import data using kibana

access kibana

```
http://localhost:5601/
```

in home page import data, click on `Upload a file`

1.new index name is: amazon_products
[[image]]

2. drag and drop the `amazon_products_cleaned.csv` located inside `cleaned_dataset` folder

[[image]]

3. In `addvanced options` replace the generated mapping by the following

[[image]]

```json
{
  "properties": {
    "actual_price": {
      "type": "double"
    },
    "discount_price": {
      "type": "double"
    },
    "image": {
      "type": "keyword",
      "index": false
    },
    "link": {
      "type": "keyword",
      "index": false
    },
    "main_category": {
      "type": "keyword"
    },
    "name": {
      "type": "text",
      "fields": {
        "keyword": { "type": "keyword" }
      }
    },
    "no_of_ratings": {
      "type": "long"
    },
    "ratings": {
      "type": "double"
    },
    "sub_category": {
      "type": "keyword"
    }
  }
}
```

4.click `Import`

The end of the import show the following screen.
[[result_import]]

## Step 4: test queries in kibabana Dev Tools

click on `menu` and select in `Management` section `Dev Tools`.

## Product related queries

##### 3.1. Get first 5 documents

```json
GET amazon_products/_search
{
  "size": 5
}
```

##### 3.2.Search by product name

```json
GET amazon_products/_search
{
  "query": {
    "match": {
      "name": "wireless headphones"
    }
  }
}
```

## Category related queries

##### 3.3.Aggregation on main category

```json
GET amazon_products/_search
{
  "size": 0,
  "aggs": {
    "main_category_details": {
      "terms":
        {
          "field": "main_category",
          "size": 20
        }
    }
  }
}
```

##### 3.4.Filter by exact main category

```json
GET amazon_products/_search
{
  "query": {
    "term": {
      "main_category": "car & motorbike"
    }
  }
}
```

## Prices related Queries

##### 3.5. Statitics on actual_price

```json
GET amazon_products/_search
{
  "size": 0,
  "aggs": {
    "all_stats_actual_price": {
      "stats": {
        "field": "actual_price"
      }
    }
  }
}
```

##### 3.6.Products discount_price within a price range

```json
GET amazon_products/_search
{
  "query": {
    "range": {
      "discount_price": {
        "gte": 500,
        "lte": 2000
      }
    }
  },
  "_source": ["name", "discount_price", "main_category"],
  "size": 10
}

```

##### 3.7.Average discount % for each main category.

```json
GET amazon_products/_search
{
  "size": 0,
  "aggs": {
    "discount_by_category": {
      "terms": {
        "field": "main_category",
        "size": 20
      },
      "aggs": {
        "avg_discount": {
          "avg": {
            "script": {
              "source": "if (doc['actual_price'].size() != 0 && doc['discount_price'].size() != 0 && doc['actual_price'].value > 0) { return (1 - doc['discount_price'].value / doc['actual_price'].value) * 100 } else { return null }"
            }
          }
        }
      }
    }
  }
}
```

## Ratings related queries

##### 3.8.Top 10 highest-rated products

```json
GET amazon_products/_search
{
  "sort": [
    { "ratings": { "order": "desc" } }
  ],
  "size": 10
}
```

##### 3.9.Group products by category and shows their average ratings.

```json
GET amazon_products/_search
{
  "size": 0,
  "aggs": {
    "avg_rating_by_category": {
      "terms": {
        "field": "main_category",
        "size": 20
      },
      "aggs": {
        "avg_rating": {
          "avg": {
            "field": "ratings"
          }
        }
      }
    }
  }
}

```

##### 3.10.Top 10 products by number of reviews

```json
GET amazon_products/_search
{
  "sort": [
    { "no_of_ratings": { "order": "desc" } }
  ],
  "_source": ["name", "main_category", "ratings", "no_of_ratings"],
  "size": 10
}
```

##### 3.11.Find "High-Speed HDMI Cable" under price 50 with rating ≥ 4, sorted by best rating

```json
GET amazon_products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "High-Speed HDMI Cable" } },
        { "term": { "main_category": "tv, audio & cameras" } }
      ],
      "filter": [
        { "range": { "discount_price": { "lte": 60 } } },
        { "range": { "ratings": { "gte": 4.0 } } }
      ]
    }
  },
  "sort": [
    { "ratings": { "order": "desc" } }
  ],
  "size": 10
}
```
