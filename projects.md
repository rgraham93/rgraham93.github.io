---
layout: page
title: Projects
permalink: /projects/
---

### Automated Testing ###

#### Chart Retrieval Automated Test Cases ####

The chart retrieval process in healthcare is the method of accessing, obtaining, and reviewing patient medical records. It’s an essential step for ensuring quality care, conducting audits, supporting billing, and meeting regulatory compliance.

Chart retrieval typically begins with a request from a healthcare provider, insurance company, or third-party auditor seeking access to patient records. These requests are often made to verify claims, conduct health audits, or gather data for quality improvement initiatives. Requests are submitted via a 'chase list,' a text file containing a list of patients, demographic information, and the start and end dates of services. To ensure that requests were created and processed successfully, I developed tests to verify that the requests were correctly inserted into the PostgreSQL database and that their status values were updated to indicate successful processing.

Code snippet of a Pytest fixture that uploads a test chase list to an AWS S3 bucket:

```python
@pytest.fixture
def upload_text_file_to_s3_bucket(fig, request):
    params = request.param

    s3 = boto3.client("s3")
    s3.upload_file(
        Filename=f"data/{params[0]}",
        Bucket=f"test-file-import-{fig['env']}",
        Key=f"{params[1]}/import/{params[0]}",
    )
    # Wait for request to process
    time.sleep(3)
```

Code snippet of a Pytest fixture that returns the status of the request in the PostgreSQL database:

```python
@pytest.fixture
def get_status_of_request_from_db(get_postgresql_db_conn, request):
    request_id = request.param

    with get_postgresql_db_conn.cursor() as cur:
        cur.execute(f"select * from customerrequest cr where cr.requestid = '{request_id}'")
        row = cur.fetchone()
        status = row[2]

    return status
```

Code snippet of a test case written in Pytest that asserts that the expected status was reached:

```python
@pytest.mark.regression
@pytest.mark.parametrize(
    "upload_text_file_to_s3_bucket, get_status_of_request_from_db",
    [
        (
            (
                "customer_request.txt",
                "customer_bucket",
            ),
            "A000001",
        )
    ],
    indirect=["upload_text_file_to_s3_bucket", "get_status_of_request_from_db"],
)
def test_status_for_request_is_processed_successfully(
    fig,
    upload_text_file_to_s3_bucket,
    get_status_of_request_from_db,
):
    assert get_status_of_request_from_db == "ProcessedSuccessfully", "Expected status was not returned."
```

---

### Performance Testing ###

#### Grafana k6 Performance Testing Repository ####

Independently implemented performance testing using Grafana k6 to help development teams optimize performance in early stages and resolve issues more quickly. The repository included performance tests for multiple API endpoints, enabling easy execution of load, stress, spike, and breakpoint tests. Configurations for each test type were readily available and could be executed seamlessly either through GitHub Actions via a dropdown menu or locally using command-line arguments.

Code snippet of an API endpoint performance test written using Grafana k6 that takes in a json file and generates an xml file using data from the json file:

``` javascript
// Options would be set up through a setupOptions helper function located in the utils folder
const optionsSetup = setupOptions(WorkloadConfig, EnvConfig);
export const options = {
    scenarios: {
        generateCCD: {
            exec: "generateCCD",
            executor: "ramping-vus",
            stages: optionsSetup.options.stages,
        },
    },
    tags: optionsSetup.options.tags,
};

// Read the deidentified FHIR bundle
let fhirBundle = open("./data/fhir_bundle.json");

export function generateCCD() {
    const url = `${optionsSetup.baseUrl}/GenerateCCD/Generate`;
    const payload = JSON.stringify({
        inputType: "FHIR_R4",
        data: `${fhirBundle}`,
    });
    const params = {
        headers: {
            "Content-Type": "application/json",
        },
        // k6 has a default timeout set to 30 seconds, if the request is expected to take longer to process, you can override the default
        timeout: "600s",
    }

    const response = http.post(url, payload, params);

    // Check that the response was successful and that the CCD has expected data in the response body
    check(response, {
        "status 200 returned": (r) => r.status === 200,
        "does response body include ccdaString": (r) => r.body.includes("ccdaString")
    });
}

```

Another exciting part of this project was implementing a hybrid performance test using the k6 browser module. A hybrid performance test is an alternative approach to browser-based load testing that’s much less resource-intensive is combining a small number of virtual users for a browser test with a large number of virtual users for a protocol-level test ([Hybrid performance test](https://grafana.com/docs/k6/latest/using-k6-browser/recommended-practices/hybrid-approach-to-performance/)). Scaffolding for an example hybrid performance test can be found in my k6 repository here [GitHub link](https://github.com/rgraham93/k6_portfolio/tree/main/browser/customer-portal).