.. _firmware-azure-building-github-actions:

Building the project using GitHub Actions
#########################################

You can use GitHub Actions (which is free for open-source projects) to build the application.
Using the `provided workflow <https://github.com/NordicSemiconductor/asset-tracker-cloud-firmware-azure/blob/saga/.github/workflows/build-and-release.yaml>`_, you can quickly set up the environment for building your application using a fork.

After you have forked the repository, `enable GitHub Actions <https://help.github.com/en/github/automating-your-workflow-with-github-actions/about-github-actions#requesting-to-join-the-limited-public-beta-for-github-actions>`_ and locate the :guilabel:`Actions` tab in your repository, which lists the Action runs.

Configuration for firmware connecting to the nRF Asset Tracker for Azure
========================================================================

Navigate to the settings of the repository and configure the secret ``AZURE_IOT_HUB_DPS_ID_SCOPE`` and assign the DPS ID scope of your Azure IoT Hub Device Provisioning to it.

To retrieve the value for the Azure IoT DPS ID scope secret, use the following command:

   .. code-block:: bash

      node cli info -o iotHubDpsIdScope

Commit a change to your repository and the GitHub Action will build the application and attach the HEX files to a release.
