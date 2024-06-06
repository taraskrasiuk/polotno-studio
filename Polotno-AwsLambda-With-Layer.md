## Polotno-node, AWS lambda with Layer configuration

In order to run the **polotno-node** in AWS Lambda, it's recommended to use a [Lambda Layer configurations](https://docs.aws.amazon.com/lambda/latest/dg/chapter-layers.html) which will manage the dependencies, like **chronium**.
In addition, the Lambda's configuration needs to be changed as well. It's recommeded to change the **timeout** to 30 seconds instead of default value (3 seconds) and **Memory limit** increase to minimum 512 Mb instead of 128 mb ( default value ).

Dependencies:

- @sparticuz/chromium
- puppeteer-core
- polotno-node

Pre-requirements:

- The chronium and puppeteer versions need to be satisfied. Please check this [document](https://pptr.dev/supported-browsers).
- The **minumum Memory limit** needs to be ```512 Mb``` for AWS Lambda. Default is ```128 Mb```.
- The **timeout** should be increased from default 3 seconds to higher value, for example up to 30 seconds. It depends on the Memory limit. For instance, if the memory limit was set to 1024, the polotno process might take ~15 secs, if 512 mb, ~30sec.

==========================================

How to create a configuration layer with a dependency on ```chronium``` ?

1. Create a ```.zip``` file from a chronium project. For example:
```
mkdir chronium-112 && cd chronium-112

npm init -y

npm install @sparticuz/chromium@112.0

zip -r chronium.zip ./*
```

2. Go to AWS console then open a Lambda section and click on ```Layers```.
3. Following the [documentation](https://docs.aws.amazon.com/lambda/latest/dg/chapter-layers.html) create a Layer with a chrnioum dependency by uploading a zip file. Keep in mind that environment like ```nodejs18.x``` should match between layer and function.

> Approximatelly, the size of the zip, will be around ```73 Mb```, so in order to upload, the S3 service will be needed.

4. Finally, open the Lambda function, select a ```Code``` section, at the bottom click on ```Add Layer``` and select a created layer.

==========================================

How to run the full example with aws-cli ?

- Create a Lambda configuration layer, and upload a **.zip** archive which stores a **chronium** npm or git package.
```
export CH_VERSION=112.0
export BUCKET=example-bucket
export AWS_ACC_ID=000000000000

// create a chronium project for Layer

mkdir chronium-layer

cd chronium-layer

npm init -y

npm install @sparticuz/chromium@${CH_VERSION}

zip -r chronium.zip ./*

aws s3 cp chronium.zip s3://${BUCKET}/lambda-layers/chronium-${CH_VERSION}.zip

aws lambda publish-layer-version \
--layer-name chronium-polotno \
--content "S3Bucket=${BUCKET},S3Key=lambda-layers/chronium-{CH_VERSION}.zip" \
--description Chronium-dev-${CH_VERSION} \
--compatible-architectures x86_64 \
--compatible-runtimes nodejs18.x
```

- Create a JS file for a Lambda's handler
```
// create a folder
mkdir handler && cd handler

// init a node project
npm init -y

// install dependencies
npm install --save polotno-node puppeteer-core@19.8.0

// create a file handler
touch index.mjs

// add code example
echo "
import chromium from "@sparticuz/chromium";
import puppeteer from "puppeteer-core";
import { createInstance } from "polotno-node";


export const handler = () => {
  const browser = await puppeteer.launch({
    args: [
      ...chromium.args,
      "--no-sandbox",
      "--hide-scrollbars",
      "--disable-web-security",
      "--allow-file-access-from-files",
      "--disable-dev-shm-usage",
    ],
    defaultViewport: chromium.defaultViewport,
    executablePath: await chromium.executablePath(),
    headless: true,
    ignoreHTTPSErrors: true,
  });

  const polotnoInstance = await createInstance({
    key: process.env.POLOTNO_API_KEY,
    browser,
  });

  const body = polotnoInstance.jsonToImageBase64(event.json)
  
  return {
    StatusCode: 200,
    headers: {
      'Content-Type': 'image/png'
    },
    body,
  }

};
" >> index.mjs

// create file with variables
echo '{
    "Variables": {
      "POLOTNO_API_KEY": "your-api-key"
    }
  }
' >> vars.json

// archive the folder
zip -r index.zip ./*

// copy to S3
aws s3 cp index.zip s3://${BUCKET}/polotno-handler.zip

// crete a trust policy
echo '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
' >> trustPolicy.json


// create roles and policies
aws iam create-role --role-name lambda-polotno-role --assume-role-policy-document file://trustPolicy.json
aws iam attach-role-policy --role-name lambda-polotno-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws lambda create-function --function-name lambda-polotno \
--runtime nodejs18.x \
--zip-file fileb://index.zip \
--layers "arn:aws:lambda:eu-central-1:${AWS_ACC_ID}:layer:chronium-polotno:1" \
--handler index.handler \
--environment file://vars.json \
--role "arn:aws:iam::${AWS_ACC_ID}:role/lambda-polotno-role" \
--timeout 60 \
--memory-size 512

// try it
aws lambda invoke --function-name lambda-polotno --cli-binary-format raw-in-base64-out --payload file://polotno-project-json.json image
```
