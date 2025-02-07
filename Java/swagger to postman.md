## Swagger to Postman Collection Automation

### Install the Converter CLI

Install node version followed by Converter CLI

```bash
sudo npm install -g openapi-to-postmanv2
```

### Convert Swagger/OpenAPI File to Postman Collection

Run the following command to generate a Postman collection:

#### Swagger file to Postman collection

```bash
openapi2postmanv2 -s swagger.json -o postman_collection.json -p
```

#### Swagger URL to Postman Collection 
curl "http://localhost:8080/v3/api-docs" -o swagger.json openapi2postmanv2 -s swagger.json -o postman_collection.json -p


- `-s` specifies the source Swagger/OpenAPI file.
- `-o` specifies the output file for the generated Postman collection.
- `-p` formats the output in a readable JSON format.

### Jenkins Pipeline for Swagger to Postman Conversion

#### Scripted Pipeline

```groovy
node {
    stage('Checkout') {
        checkout scm
    }

    stage('Install Dependencies') {
        sh 'npm install -g openapi-to-postmanv2'
    }

    stage('Convert API') {
        sh 'curl "http://localhost:8080/v3/api-docs" -o swagger.json openapi2postmanv2 -s swagger.json -o postman_collection.json -p'
    }
    
    stage('Archive Output') {
        archiveArtifacts artifacts: 'postman_collection.json', fingerprint: true
    }
}
```

#### Declarative Pipeline

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Install Dependencies') {
            steps {
                sh 'npm install -g openapi-to-postmanv2'
            }
        }
        stage('Convert API') {
            steps {
               sh 'curl "http://localhost:8080/v3/api-docs" -o swagger.json openapi2postmanv2 -s swagger.json -o postman_collection.json -p'
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'postman_collection.json', fingerprint: true
        }
    }
}
```
