# Elastic Task
## Overview
This document outlines the steps for converting a CSV file to JSON format, uploading the JSON data into an Elasticsearch index using the Bulk API, and then downloading the data from Elasticsearch using Python scripts.
## File Descriptions

### 1. `csv_to_json.py`
This script converts a CSV file into a JSON file compatible with Elasticsearch's Bulk API.

#### Inputs
- `output.csv`: The input CSV file.

#### Outputs
- `output.json`: The JSON file formatted for Elasticsearch Bulk API.

#### Code
```python
import csv
import json

input_file = 'output.csv'
output_file = 'output.json'  

index_name = "task"  

with open(input_file, 'r') as csv_file, open(output_file, 'w') as json_file:
    csv_reader = csv.DictReader(csv_file)
    for idx, row in enumerate(csv_reader):
        action = {"index": {"_index": index_name, "_id": idx + 1}}
        json_file.write(json.dumps(action) + '\n')
        json_file.write(json.dumps(row) + '\n')

print(f"Converted JSON saved to {output_file}")
```

---

### 2. `bulk.py`
This script uploads the generated JSON file (`output.json`) into an Elasticsearch index using the Bulk API.

#### Code
```python
from elasticsearch import Elasticsearch

# Connect to Elasticsearch
es = Elasticsearch("http://localhost:9200")

# Read the JSON bulk file
with open("output.json", "r") as file:
    bulk_data = file.read()

# Upload data using Bulk API
response = es.bulk(body=bulk_data)
print(response)
```

---

### 3. `download_index.py` (Multithreaded)
This script retrieves all data from the Elasticsearch index using a multithreaded approach and saves it as a JSON file.

#### Outputs
- `full_download.json`: The downloaded data from Elasticsearch.

#### Code
```python
import time
import json
from elasticsearch import Elasticsearch
from concurrent.futures import ThreadPoolExecutor, as_completed


def fetch_scroll(scroll_id, scroll_time):
    # Connect to Elasticsearch
    es = Elasticsearch("http://localhost:9200")
    response = es.scroll(scroll_id=scroll_id, scroll=scroll_time)
    return response["hits"]["hits"]

def main():

    start_time = time.time()

    # Connect to Elasticsearch
    es = Elasticsearch("http://localhost:9200")

    # Initialize the scroll API
    response = es.search(index="task", body={"query": {"match_all": {}}}, scroll="2m", size=100)
    scroll_id = response["_scroll_id"]
    documents = response["hits"]["hits"]

    # Thread pool executor for multithreading
    scroll_time = "2m"
    max_workers = 10  # Adjust the number of threads as needed
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        # List to store futures
        futures = []
        while len(response["hits"]["hits"]) > 0:
            # Submit a thread for each scroll batch
            futures.append(executor.submit(fetch_scroll, scroll_id, scroll_time))
            response = es.scroll(scroll_id=scroll_id, scroll=scroll_time)

        # Wait for all threads to complete and collect results
        for future in as_completed(futures):
            documents.extend(future.result())

    # Save the results to a file
    with open("full_download.json", "w") as file:
        json.dump([doc["_source"] for doc in documents], file, indent=4)


    end_time = time.time()

    # Calculate and print the elapsed time
    elapsed_time = end_time - start_time
    print(f"All data saved to full_download.json")
    print(f"Time taken: {elapsed_time:.2f} seconds")

if __name__ == "__main__":
    main()

```
**output**: `Total download time: 0.80 seconds`

---
Here is the bar chart comparing the download times with and without threading
![](https://i.imgur.com/cgByQAX.png)

## Workflow Summary
1. **Convert CSV to JSON**:
   - Execute `csv_to_json.py` to generate the `output.json` file.
2. **Upload JSON to Elasticsearch**:
   - Run `bulk.py` to push the data into Elasticsearch.
3. **Download Data from Elasticsearch**:
   - Use `download_index.py` to retrieve all data from the Elasticsearch index and save it to `full_download.json`.
