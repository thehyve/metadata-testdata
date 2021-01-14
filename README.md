# Testdata generator for metadata in Fairspace

## Dependencies

Requires Python 3.7 or newer.
Preferably run within a virtual environment:
```
python3 -m venv venv/
source venv/bin/activate
```
Install:
```
pip install .
```

## Testdata generation

Please check if the settings in [`upload_test_data.py`](metadata_scripts/upload_test_data.py) are adequate.
Do not forget to reinstall the package after any changes.

Run the script:
```
upload_test_data
```
This uploads 30,000 subjects, 60,000 tumor pathology events,
120,000 samples and creates 10, each containing 100
directories with 1,500 files.

Or run with different parameters:
```python
from metadata_scripts.upload_test_data import TestData
def testrun():
    testdata = TestData()
    testdata.subject_count = 100
    testdata.event_count = 200
    testdata.sample_count = 500
    testdata.collection_count = 5
    testdata.dirs_per_collection = 5
    testdata.files_per_dir = 10
    testdata.run()

testrun()
```

## Run queries

The `retrieve_view` command retrieves the first page of samples by default,
use `retrieve_view Subject` to retrieve a page of subjects, etc.

The API for retrieving pages can be used directly to specify the page and
filters:
```python
from fairspace_api.api import FairspaceApi
import json
import time

api = FairspaceApi()

filters = [{'field': 'gender', 'values': ['male']}]
start = time.time()
try:
    count = api.count('samples', filters=filters)
    print(f'Took {time.time() - start:.0f}s.')
    print(count)
    start = time.time()
    page = api.retrieve_view_page('samples', page=1, size=2, filters=filters)
    print(f'Took {time.time() - start:.0f}s.')
    print(page)
except Exception as e:
    print('Error', e)


# Retrieve first 2 female subjects for which blood samples with ChIP-on-Chip analysis are available
page = api.retrieve_view_page('Sample', page=1, size=2, include_counts=True, filters=[
  {'field': 'Subject.gender', 'values': ['Female']},
  {'field': 'Sample.nature', 'values': ['Blood']},
  {'field': 'Collection.analysisType', 'values': ['ChIP-on-Chip']}
])
print(json.dumps(page.__dict__, indent=2))
```

SPARQL queries:
```python
from fairspace_api.api import FairspaceApi
api = FairspaceApi()
query = """
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX curie: <https://institut-curie.org/ontology#>
PREFIX fs:    <https://fairspace.nl/ontology#>

SELECT ?sample ?sampleTopography ?sampleNature ?sampleOrigin ?tumorCellularity ?event ?tumorTopography ?morphology ?eventType
    ?laterality ?tumorGradeType ?tumorGradeValue ?tnmType ?tnmT ?tnmN ?tnmM ?ageAtDiagnosis ?subject ?gender
    ?species ?ageAtLastNews ?ageAtDeath
WHERE {
    OPTIONAL {?sample curie:topography ?sampleTopography }
    OPTIONAL {?sample curie:isOfNature ?sampleNature }
    OPTIONAL {?sample curie:hasOrigin ?sampleOrigin }
    OPTIONAL {?sample curie:tumorCellularity ?tumorCellularity }
    
    ?sample curie:subject ?subject .
    OPTIONAL {?subject curie:isOfSpecies ?species}
    OPTIONAL {?subject curie:gender ?gender}
    OPTIONAL {?subject curie:ageAtLastNews ?ageAtLastNews}
    OPTIONAL {?subject curie:ageAtDeath ?ageAtDeath}

    OPTIONAL {
        ?sample curie:diagnosis ?event .
        
        OPTIONAL { ?event curie:tumorMorphology ?morphology }
        OPTIONAL { ?event curie:eventType ?eventType }
        OPTIONAL { ?event curie:topography ?tumorTopography }
        OPTIONAL { ?event curie:tumorLaterality ?laterality }
        OPTIONAL { ?event curie:ageAtDiagnosis ?ageAtDiagnosis }
        OPTIONAL { ?event curie:tumorGradeType ?tumorGradeType }
        OPTIONAL { ?event curie:tumorGradeValue ?tumorGradeValue }
        OPTIONAL { ?event curie:tnmType ?tnmType }
        OPTIONAL { ?event curie:tnmT ?tnmT }
        OPTIONAL { ?event curie:tnmN ?tnmN }
        OPTIONAL { ?event curie:tnmM ?tnmM }
    }

    ?sample a curie:BiologicalSample .
    FILTER NOT EXISTS { ?sample fs:dateDeleted ?anyDateDeleted }
}
LIMIT 10
"""
results = api.query_sparql(query)
sample_ids = [sample['sample']['value'] for sample in results['results']['bindings']]
print(len(sample_ids), 'samples,', len(set(sample_ids)), 'unique:', set(sample_ids))
```
