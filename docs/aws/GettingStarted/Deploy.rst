.. _aws-getting-started-deploy:

Install the nRF Asset Tracker into your AWS account
###################################################

The following commands set up the necessary resources in your AWS account:

.. code-block:: bash

    # ~/nrf-asset-tracker/aws

    # One-time operation to support large CloudFormation templates in CDK
    npx cdk -a 'node dist/cdk/cloudformation-sourcecode.js' bootstrap aws://`aws sts get-caller-identity | jq -r '.Account' | tr -d '\n'`/${AWS_REGION}

    # Create the S3 Bucket for publishing the lambdas
    npx cdk -a 'node dist/cdk/cloudformation-sourcecode.js' deploy
    
    # Deploy the example (see the note below)
    #   It will prompt:
    #     Do you wish to deploy these changes (y/n)?
    #   twice (the first is for the main stack, and the second is when deploying the web application stack)
    #   Both need to be confirmed with 'y'
    npx cdk deploy --all

    # This is a fix for a bug with AWS CloudFormation and HTTP APIs
    # See https://github.com/bifravst/aws/issues/455
    node dist/cdk/helper/addFakeRoute.js

The initial deployment will take a few minutes because it sets up a `CloudFront <https://aws.amazon.com/cloudfront/>`_ distribution for the web application.

.. note::

    The AWS CDK provides a list of permission changes to your account, and you need to review them carefully whenever you make changes to the setup.
    However, this step is not mandatory and you can safely skip it, because you are deploying in a blank account.
    To skip this step in future, run ``npx cdk deploy --all --require-approval never``.
    Do this only in standalone accounts for development purposes.