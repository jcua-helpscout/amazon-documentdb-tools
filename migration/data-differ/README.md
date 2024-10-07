# Amazon DocumentDB DataDiffer Tool

** Note: The source is a MongoDB 3.4.24 and the target is a AWS DocumentDB 3.6.0.

The purpose of the DataDiffer tool is to facilitate the validation of data consistency by comparing two collections, making it particularly useful in migration scenarios.
This tool performs the following checks:

- Document existence check: It reads documents in batches from the source collection and checks for their existence in the target collection. If there is a discrepancy, the tool attempts will identify and report the missing documents.
- Index Comparison: examines the indexes of the collections and reports any differences.
- Document Comparison: each document in the collections, with the same _id, is compared using the DeepDiff library. This process can be computationally intensive, as it involves scanning all document fields. The duration of this check depends on factors such as document complexity and the CPU resources of the machine executing the script.

## Prerequisites:

 - Python 3.8
 - Modules: pymongo, deepdiff, tqdm
 - DocumentDB - disable the TLS
```
  "/opt/homebrew/Cellar/python@3.8/3.8.18_2/bin/python3.8" -m venv venv_3_8
  . venv_3_8/bin/activate
  pip freeze    # should be empty for now
  pip install pymongo==3.6.0 deepdiff==8.0.1 tqdm==4.66.5 orderly-set==5.2.2
  wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```
Note: Refer to the DeepDiff [documentation](https://zepworks.com/deepdiff/current/optimizations.html) for potential optimizations you may try out specifically for your dataset.

## How to use

1. Clone the repository and go to the tool folder:
```
git clone https://github.com/jcua-helpscout/amazon-documentdb-tools
cd amazon-documentdb-tools/migration/data-differ/
```

2. Run the data-differ.py tool, which accepts the following arguments:

```
python3 data-differ.py --help
usage: data-differ.py [-h] [--batch_size BATCH_SIZE] [--output_file OUTPUT_FILE] [--check_target CHECK_TARGET] --source-uri SOURCE_URI --target-uri TARGET_URI --source-db SOURCE_DB --target-db TARGET_DB --source-coll SOURCE_COLL --target-coll TARGET_COLL [--sample_size_percent SAMPLE_SIZE_PERCENT] [--sampling_timeout_ms SAMPLING_TIMEOUT_MS]

Compare two collections and report differences.

options:
  -h, --help            show this help message and exit
  --batch_size BATCH_SIZE
                        Batch size for bulk reads (optional, default: 100)
  --output_file OUTPUT_FILE
                        Output file path (optional, default: differences.txt)
  --check_target CHECK_TARGET
                        optional, Check if extra documents exist in target database
  --source-uri SOURCE_URI
                        Source cluster URI (required)
  --target-uri TARGET_URI
                        Target cluster URI (required)
  --source-db SOURCE_DB
                        Source database name (required)
  --target-db TARGET_DB
                        Target database name (required)
  --source-coll SOURCE_COLL
                        Source collection name (required)
  --target-coll TARGET_COLL
                        Target collection name (required)
  --sample_size_percent SAMPLE_SIZE_PERCENT
                        optional, if set only samples a percentage of the documents
  --sampling_timeout_ms SAMPLING_TIMEOUT_MS
                        optional, override the timeout for returning a sample of documents when using the --sample_size_percent argument
```

## Example usage:
Connect to a standalone MongoDB instance as source and to a Amazon DocumentDB cluster as target.

From the source uri, compare the collection *mysourcecollection* from database *mysource*, against the collection *mytargetcollection* from database *mytargetdb* in the target uri.

```
python3 data-differ-args.py \
--source-uri "mongodb://user:password@mongodb-instance-hostname:27017" \
--target-uri "mongodb://user:password@target.cluster.docdb.amazonaws.com:27017/?ssl=true&ssl_ca_certs=global-bundle.pem&replicaSet=rs0&readPreference=secondaryPreferred&retryWrites=false" \
--source-db mysourcedb \
--source-coll mysourcecollection \
--target-db mytargetdb \
--target-coll mytargetcollection \
--sample_size_percent 1
```

For more information on the connection string format, refer to the [documentation](https://www.mongodb.com/docs/manual/reference/connection-string/).

## Sampling
For large databases it might be unfeasible to compare every document as:
* It takes a long time to compare every document.
* Reading every document from a large busy database could have a performance impact.

If you use the `--sample_size_percent` option you can pass in a percentage of
documents to sample and compare.

E.g. `--sample_size_percent 1` would sample 1% of the documents in the source
database and compare them to the target database.

Under the hood this uses the [MongoDB `$sample` operator](https://www.mongodb.com/docs/manual/reference/operator/aggregation/sample/)
You should read the documentation on how that behaves on your version of MongoDB
when the percentage to sample is >= 5% before picking a percentage to sample.

The default timeout for retriving a sample of documents is `500ms`, if this is
not long enough you can adjust it with the `--sampling_timeout_ms` argument.
For example `--sample_timeout_ms 600` would increase the timeout to `600ms`.
