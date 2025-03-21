---
title: Enhance security posture with security recommendations - Microsoft Defender for IoT
description: Learn about how to find security recommendations for devices detected by Microsoft Defender for IoT.
ms.date: 12/12/2022
ms.topic: how-to
---

# Enhance security posture with security recommendations

Use Microsoft Defender for IoT's security recommendations to enhance your network security posture across unhealthy devices in your network. Lower your attack surface by creating actionable, prioritized mitigation plans that address the unique challenges in OT/IoT networks.

> [!IMPORTANT]
> The **Recommendations** page is currently in **PREVIEW**. See the [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/) for additional legal terms that apply to Azure features that are in beta, preview, or otherwise not yet released into general availability.

## View security recommendations

View all current recommendations for your organization on the Defender for IoT **Recommendations** page on the Azure portal. For example:

:::image type="content" source="media/recommendations/recommendations.png" alt-text="Screenshot of the Recommendations page on the Azure portal." lightbox="media/recommendations/recommendations.png":::

The **Active recommendations** widget indicates the number of recommendations that represent actionable steps you can currently take on unhealthy devices. We recommend reviewing unhealthy devices regularly, taking recommended actions, and keeping the number of active recommendations as low as possible.

Recommendations are shown in a grid with details in the following columns:

|Column name  |Description  |
|---------|---------|
|**Severity**     | Indicates the urgency of the suggested mitigation step.         |
|**Name**     |  The recommendation's name, which indicates a summary of the suggested mitigation step.  |
|**Unhealthy devices**     | The number of detected devices where the recommended step is relevant.        |
|**Healthy devices**     |   The number of detected devices where the recommended step is covered and no action is required.      |
|**Last update time**     | The last time the recommendation was triggered on a detected device.        |

Do either of the following to modify the recommendation data listed:

- Select :::image type="icon" source="media/how-to-manage-device-inventory-on-the-cloud/edit-columns-icon.png" border="false"::: **Edit columns** to add or remove columns from the grid.
- Filter the list by entering a keyword from the recommendation name in the **Search** box, or select **Add filter** to filter the grid by any of the recommendation columns.

To export a CSV file of all recommendations for your network, select :::image type="icon" source="media/how-to-manage-device-inventory-on-the-cloud/export-button.png" border="false" :::**Export**.

## View recommendation details

Select a specific recommendation in the grid to drill down for more details. The recommendation name is shown as the page's title, with details with the recommendation's severity, number of unhealthy devices detected, and last update date and time in widgets on the left.

The left pane also shows the following information:

- **Description**: More context for the recommended mitigation step
- **Remediation steps**: The full list of mitigation steps recommended for unhealthy devices

Switch between the **Unhealthy devices** and **Healthy devices** tabs to review the statuses of detected devices in your network for the selected recommendation.

For example:

:::image type="content" source="media/release-notes/recommendations.png" alt-text="Screenshot of the Review PLC operating mode recommendation page." lightbox="media/release-notes/recommendations.png":::

### View recommendation details by device

You might want to review all recommendations for a specific device in order to handle them all together.

Recommendations are also listed on the **Device details** page for each detected device, accessed either from the [**Device inventory** page](how-to-manage-device-inventory-for-organizations.md#view-the-device-inventory), or from the list of healthy or unhealthy devices on a recommendation details page.

On a device details page, select the **Recommendations** tab to view a list of security recommendations specific for the selected device.

For example:

:::image type="content" source="media/recommendations/recommendations-device-details.png" alt-text="Screenshot of the Recommendations tab on a device details page." lightbox="media/recommendations/recommendations-device-details.png":::

## Supported security recommendations

The following recommendations are displayed for devices detected by OT and Enterprise IoT network sensors:

|Name  |Description  |
|---------|---------|
| **OT network sensors** | |
|**Review PLC operating mode**     | Devices with this recommendation are found with PLCs set to unsecure operating mode states. <br><br>We recommend setting PLC operating modes to the **Secure Run** state if access is no longer required to the PLC to reduce the threat of malicious PLC programming.        |
|**Review unauthorized devices**     | Devices with this recommendation must be identified and authorized as part of the network baseline. <br><br>We recommend taking action to identify any indicated devices. Disconnect any devices from your network that remain unknown even after investigation to reduce the threat of rogue or potentially malicious devices.        |
| **Enterprise IoT network sensors** | |
| **Disable insecure administration protocol**| Devices with this recommendation are exposed to malicious threats because they use Telnet, which isn't a secured and encrypted communication protocol. <br><br>We recommend that you switch to a more secure protocol, such as SSH, disable the server altogether, or apply network access restrictions.|

Other recommendations you may see in the **Recommendations** page are relevant for the  [Defender for IoT micro agent](/azure/defender-for-iot/device-builders/).

## Next steps

> [!div class="nextstepaction"]
> [View the device inventory](how-to-manage-device-inventory-for-organizations.md#view-the-device-inventory)

