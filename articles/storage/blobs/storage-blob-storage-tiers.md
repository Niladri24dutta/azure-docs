---
title: Azure hot, cool, and archive storage for blobs | Microsoft Docs
description: Hot, cool, and archive storage for Azure Blob storage accounts.
services: storage
documentationcenter: ''
author: michaelhauss
manager: vamshik
editor: tysonn

ms.assetid: eb33ed4f-1b17-4fd6-82e2-8d5372800eef
ms.service: storage
ms.workload: storage
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: get-started-article
ms.date: 06/05/2017
ms.author: mihauss

---
# Azure Blob Storage: Hot, cool, and archive (preview) storage tiers

## Overview

Azure Storage offers three storage tiers for Blob object storage so that you can store your data most cost-effectively depending on how you use it. The Azure **hot storage tier** is optimized for storing data that is accessed frequently. The Azure **cool storage tier** is optimized for storing data that is infrequently accessed and stored for at least 30 days. The Azure **archive storage tier** (preview) is optimized for storing data that is rarely accessed and stored for at least 180 days with flexible latency requirements (on the order of hours). The archive storage tier is only available at the blob level and not at the storage account level. Data in the cool storage tier can tolerate slightly lower availability, but still requires high durability and similar time-to-access and throughput characteristics as hot data. For cool data, a slightly lower availability SLA and higher access costs compared to hot data are acceptable trade-offs for lower storage costs. Archive storage is offline and offers the lowest storage costs but also the highest access costs.

Today, data stored in the cloud is growing at an exponential pace. To manage costs for your expanding storage needs, it's helpful to organize your data based on attributes like frequency-of-access and planned retention period to optimize costs. Data stored in the cloud can be different in terms of how it is generated, processed, and accessed over its lifetime. Some data is actively accessed and modified throughout its lifetime. Some data is accessed frequently early in its lifetime, with access dropping drastically as the data ages. Some data remains idle in the cloud and is rarely, if ever, accessed once stored.

Each of these data access scenarios benefits from a different storage tier that is optimized for a particular access pattern. With hot, cool, and archive storage tiers, Azure Blob storage addresses this need for differentiated storage tiers with separate pricing models.

## Blob storage accounts

**Blob storage accounts** are specialized storage accounts for storing your unstructured data as blobs (objects) in Azure Storage. With Blob storage accounts, you can now choose between hot and cool storage tiers at account level, or hot, cool, and archive tiers at the blob level based on access patterns. Store rarely, infrequently, and frequently accessed data in the hot, cool, and archive storage tiers respectively to optimize costs. Blob storage accounts are similar to your existing general-purpose storage accounts and share all the great durability, availability, scalability, and performance features that you use today, including 100 percent API consistency for block blobs and append blobs.

> [!NOTE]
> Blob storage accounts support only block and append blobs, and not page blobs.

Blob storage accounts expose the **Access Tier** attribute at the account level, which specify the default storage account tier as **Hot** or **Cool**. The default storage account tier is applied to any blob that does not have an explicit tier set at the blob level. If there is a change in the usage pattern of your data, you can also switch between these storage tiers at any time. The **archive tier** (preview) can only be applied at the blob level.

> [!NOTE]
> Changing the storage tier may result in additional charges. See the [Pricing and billing](#pricing-and-billing) section for more details.

### Hot access tier

Hot storage has higher storage costs than cool and archive storage, but the lowest access costs. Example usage scenarios for the hot storage tier include:

* Data that is in active use or expected to be accessed (read from and written to) frequently.
* Data that is staged for processing and eventual migration to the cool storage tier.

### Cool access tier

Cool storage tier has lower storage costs and higher access costs compared to hot storage. This tier is intended for data that will remain in the cool tier for at least 30 days. Example usage scenarios for the cool storage tier include:

* Short-term backup and disaster recovery datasets.
* Older media content not viewed frequently anymore but is expected to be available immediately when accessed.
* Large data sets that need to be stored cost effectively while more data is being gathered for future processing. (*For example*, long-term storage of scientific data, raw telemetry data from a manufacturing facility)

### Archive access tier (preview)

Archive storage has the lowest storage cost and higher data retrieval costs compared to hot and cool storage. This tier is intended for data that can tolerate several hours of retrieval latency and will remain in the archive tier for at least 180 days.

While a blob is in archive storage, it is offline and cannot be read (except the metadata, which is online and available), copied, overwritten, or modified. Nor can you take snapshots of a blob in archive storage. However, you may use existing operations to delete, list, get blob properties/metadata, or change the tier of your blob.

#### Blob rehydration
To read data in archive storage, you must first change the tier of the blob to hot or cool. This process is known as rehydration and can take up to 15 hours to complete for blobs less than 50 GB. Additional time required for larger blobs varies with the blob throughput limit.

During rehydration, you may check the "archive status" blob property to confirm if the tier has changed. The status reads "rehydrate-pending-to-hot" or "rehydrate-pending-to-cool" depending on the destination tier. Upon completion, the "archive status" blob property is removed, and the "access tier" blob property reflects the hot or cool tier.  

Example usage scenarios for the archive storage tier include:

* Long-term backup, archival, and disaster recovery datasets
* Original (raw) data that must be preserved, even after it has been processed into final usable form. (*For example*, Raw media files after transcoding into other formats)
* Compliance and archival data that needs to be stored for a long time and is hardly ever accessed. (*For example*, Security camera footage, old X-Rays/MRIs for healthcare organizations, audio recordings, and transcripts of customer calls for financial services)

### Recommendations

See [About Azure storage accounts](../common/storage-create-storage-account.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json) for more information on storage accounts.

For applications requiring only block or append blob storage, we recommend using Blob storage accounts, to take advantage of the differentiated pricing model of tiered storage. However, we understand this might not be possible under certain circumstances where using general-purpose storage accounts would be the way to go, such as:

* You need to use tables, queues, or files, and want your blobs stored in the same storage account. There is no technical advantage to storing these in the same account other than having the same shared keys.

* You still need to use the Classic deployment model. Blob storage accounts are only available via the Azure Resource Manager deployment model.

* You need to use page blobs. Blob storage accounts do not support page blobs. We generally recommend using block blobs unless you have a specific need for page blobs.

* You use a version of the [Storage Services REST API](https://msdn.microsoft.com/library/azure/dd894041.aspx) that is earlier than 2014-02-14 or a client library with a version lower than 4.x, and cannot upgrade your application.

> [!NOTE]
> Blob storage accounts are currently supported in all Azure regions.


## Blob-level tiering feature (preview)

Blob-level tiering allows you to change the tier of your data at the object level using a single operation called [Set Blob Tier](/rest/api/storageservices/set-blob-tier). You can easily change the access tier of a blob among the hot, cool, or archive tiers as usage patterns change, without having to move data between accounts. All tier changes happen immediately except when a blob is rehydrating from archive. The time of the last blob tier change is exposed via the **Access Tier Change Time** attribute in the blob properties. If a blob is in the archive tier, it may not be overwritten, and therefore, uploading the same blob is not allowed in this scenario. You may overwrite a blob in hot and cool, and in this case, the new blob inherits the tier of the old blob that was overwritten.

Blobs in all three storage tiers can co-exist within the same account. Any blob that does not have an explicitly assigned tier infers the tier from the account access tier setting. If the access tier is inferred from the account, you see the **Access Tier Inferred** attribute set to “true”, and the blob **Access Tier** attribute matches the account tier. In the Azure portal, the access tier inferred property is displayed with the blob access tier (for example, Hot (inferred) or Cool (inferred)).

> [!NOTE]
> Archive storage and blob-level tiering only support block blobs. You also cannot change the tier of a block blob that has snapshots.

### Blob-level tiering billing

When a blob is moved to a cooler tier (Hot->Cool, Hot->Archive, or Cool->Archive), the operation is billed as a write into the destination tier, and the write operation (per 10,000) and data write (per GB) charges of the destination tier apply. If a blob is moved to a warmer tier (Archive -> Cool, Archive -> Hot, or Cool->Hot), the operation is billed as a read from the source tier, and the read operation (per 10,000) and data retrieval (per GB) charges of the source tier apply.

To use these features in preview, follow the instructions in the [Azure Archive and Blob-Level Tiering blog announcement](https://azure.microsoft.com/blog/announcing-the-public-preview-of-azure-archive-blob-storage-and-blob-level-tiering).

The follow lists some restrictions that apply during preview for blob-level tiering:

* Only new Blob storage accounts created in US East 2, US East, or US West after successful preview enrollment support archive storage.

* Only new Blob storage accounts created in public regions after successful preview enrollment support blob-level tiering.

* Blob-level tiering and archive storage are supported in [LRS] (../common/storage-redundancy.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json#locally-redundant-storage) storage only. [GRS](../common/storage-redundancy.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json#geo-redundant-storage) and [RA-GRS](../common/storage-redundancy.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json#read-access-geo-redundant-storage) will be supported in the future.

* You may not change the tier of a blob with snapshots.

* You may not copy or snapshot a blob in archive storage.

## Comparison of the storage tiers

The following table shows a comparison of the hot and cool storage tiers. The archive blob-level tier is in preview, so there are no SLAs for it.

| | **Hot storage tier** | **Cool storage tier** | **Archive storage tier**
| ---- | ----- | ----- | ----- |
| **Availability** | 99.9% | 99% | N/A |
| **Availability** <br> **(RA-GRS reads)**| 99.99% | 99.9% | N/A |
| **Usage charges** | Higher storage costs, lower access and transaction costs | Lower storage costs, higher access and transaction costs | Lowest storage costs, highest access and transaction costs |
| **Minimum object size** | N/A | N/A | N/A |
| **Minimum storage duration** | N/A | N/A | 180 days
| **Latency** <br> **(Time to first byte)** | milliseconds | milliseconds | < 15 hrs
| **Scalability and performance targets** | Same as general-purpose storage accounts | Same as general-purpose storage accounts | Same as general-purpose storage accounts |

> [!NOTE]
> Blob storage accounts support the same performance and scalability targets as general-purpose storage accounts. See [Azure Storage Scalability and Performance Targets](../common/storage-scalability-targets.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json) for more information.


## Pricing and billing
Blob storage accounts use a pricing model for blob storage based on the tier of each blob. When using a Blob storage account, the following billing considerations apply:

* **Storage costs**: In addition to the amount of data stored, the cost of storing data varies depending on the storage tier. The per-gigabyte cost decreases as the tier gets cooler.

* **Data access costs**: Data access charges increase as the tier gets cooler. For data in the cool and archive storage tier, you are charged a per-gigabyte data access charge for reads.

* **Transaction costs**: There is a per-transaction charge for all tiers that increases as the tier gets cooler.

* **Geo-Replication data transfer costs**: This only applies to accounts with geo-replication configured, including GRS and RA-GRS. Geo-replication data transfer incurs a per-gigabyte charge.

* **Outbound data transfer costs**: Outbound data transfers (data that is transferred out of an Azure region) incur billing for bandwidth usage on a per-gigabyte basis, consistent with general-purpose storage accounts.

* **Changing the storage tier**: Changing the account storage tier from cool to hot incurs a charge equal to reading all the data existing in the storage account. However, changing the account storage tier from hot to cool incurs a charge equal to writing all the data into the cool tier.

> [!NOTE]
> For more details on the pricing model for Blob storage accounts, see [Azure Storage Pricing](https://azure.microsoft.com/pricing/details/storage/) page. For more details on the outbound data transfer charges, see [Data Transfers Pricing Details](https://azure.microsoft.com/pricing/details/data-transfers/) page.

## Quick start

In this section, the following scenarios are demonstrated using the Azure portal:

* How to create a Blob storage account.
* How to manage a Blob storage account.

You cannot set the access tier to archive in the following examples because this setting applies to the whole storage account. Archive can only be set on a specific blob.

### Create a Blob storage account using the Azure portal

1. Sign in to the [Azure portal](https://portal.azure.com).

2. On the Hub menu, select **New** > **Data + Storage** > **Storage account**.

3. Enter a name for your storage account.

    This name must be globally unique; it is used as part of the URL used to access the objects in the storage account.  

4. Select **Resource Manager** as the deployment model.

    Tiered storage can only be used with Resource Manager storage accounts; this is the recommended deployment model for new resources. For more information, check out the [Azure Resource Manager overview](../../azure-resource-manager/resource-group-overview.md).  

5. In the Account Kind dropdown list, select **Blob Storage**.

    This is where you select the type of storage account. Tiered storage is not available in general-purpose storage; it is only available in the Blob storage type account.     

    When you select this, the performance tier is set to Standard. Tiered storage is not available with the Premium performance tier.

6. Select the replication option for the storage account: **LRS**, **GRS**, or **RA-GRS**. The default is **RA-GRS**.

    LRS = locally redundant storage; GRS = geo-redundant storage (two regions); RA-GRS is read-access geo-redundant storage (2 regions with read access to the second).

    For more details on Azure Storage replication options, check out [Azure Storage replication](../common/storage-redundancy.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json).

7. Select the right storage tier for your needs: Set the **Access tier** to either **Cool** or **Hot**. The default is **Hot**.

8. Select the subscription in which you want to create the new storage account.

9. Specify a new resource group or select an existing resource group. For more information on resource groups, see [Azure Resource Manager overview](../../azure-resource-manager/resource-group-overview.md).

10. Select the region for your storage account.

11. Click **Create** to create the storage account.

### Change the storage tier of a Blob storage account using the Azure portal

1. Sign in to the [Azure portal](https://portal.azure.com).

2. To navigate to your storage account, select All Resources, then select your storage account.

3. In the Settings blade, click **Configuration** to view and/or change the account configuration.

4. Select the right storage tier for your needs: Set the **Access tier** to either **Cool** or **Hot**.

5. Click Save at the top of the blade.

### Change the storage tier of a blob using the Azure portal

1. Sign in to the [Azure portal](https://portal.azure.com).

2. To navigate to your blob in your storage account, select All Resources, select your storage account, select your container, and then select your blob.

3. In the Blob properties blade, click the **Access Tier** dropdown menu to select the **Hot**, **Cool**, or **Archive** storage tier.

5. Click Save at the top of the blade.

> [!NOTE]
> Changing the storage tier may result in additional charges. See the [Pricing and Billing](#pricing-and-billing) section for more details.


## Evaluating and migrating to Blob storage accounts
The purpose of this section is to help users to make a smooth transition to using Blob storage accounts. There are two user scenarios:

* You have an existing general-purpose storage account and want to evaluate a change to a Blob storage account with the right storage tier.
* You have decided to use a Blob storage account or already have one and want to evaluate whether you should use the hot or cool storage tier.

In both cases, the first order of business is to estimate the cost of storing and accessing your data stored in a Blob storage account and compare that against your current costs.

## Evaluating Blob storage account tiers

In order to estimate the cost of storing and accessing data stored in a Blob storage account, you need to evaluate your existing usage pattern or approximate your expected usage pattern. In general, you want to know:

* Your storage consumption - How much data is being stored and how does this change on a monthly basis?

* Your storage access pattern - How much data is being read from and written to the account (including new data)? How many transactions are used for data access, and what kinds of transactions are they?

## Monitoring existing storage accounts

To monitor your existing storage accounts and gather this data, you can make use of Azure Storage Analytics, which performs logging and provides metrics data for a storage account. Storage Analytics can store metrics that include aggregated transaction statistics and capacity data about requests to the Blob storage service for both general-purpose storage accounts as well as Blob storage accounts. This data is stored in well-known tables in the same storage account.

For more details, see [About Storage Analytics Metrics](https://msdn.microsoft.com/library/azure/hh343258.aspx) and [Storage Analytics Metrics Table Schema](https://msdn.microsoft.com/library/azure/hh343264.aspx)

> [!NOTE]
> Blob storage accounts expose the table service endpoint only for storing and accessing the metrics data for that account.

To monitor the storage consumption for the Blob storage service, you need to enable the capacity metrics.
With this enabled, capacity data is recorded daily for a storage account's Blob service and recorded as a table entry that is written to the *$MetricsCapacityBlob* table within the same storage account.

To monitor the data access pattern for the Blob storage service, you need to enable the hourly transaction metrics at an API level. With this enabled, per API transactions are aggregated every hour, and recorded as a table entry that is written to the *$MetricsHourPrimaryTransactionsBlob* table within the same storage account. The *$MetricsHourSecondaryTransactionsBlob* table records the transactions to the secondary endpoint when using RA-GRS storage accounts.

> [!NOTE]
> In case you have a general-purpose storage account in which you have stored page blobs and virtual machine disks alongside block and append blob data, this estimation process is not applicable. This is because you have no way of distinguishing capacity and transaction metrics based on the type of blob for only block and append blobs, which can be migrated to a Blob storage account.

To get a good approximation of your data consumption and access pattern, we recommend you choose a retention period for the metrics that is representative of your regular usage and extrapolate. One option is to retain the metrics data for seven days and collect the data every week, for analysis at the end of the month. Another option is to retain the metrics data for the last 30 days and collect and analyze the data at the end of the 30-day period.

For details on enabling, collecting, and viewing metrics data, see [Enabling Azure Storage metrics and viewing metrics data](../common/storage-enable-and-view-metrics.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json).

> [!NOTE]
> Storing, accessing, and downloading analytics data is also charged just like regular user data.

### Utilizing usage metrics to estimate costs

### Storage costs

The latest entry in the capacity metrics table *$MetricsCapacityBlob* with the row key *'data'* shows the storage capacity consumed by user data. The latest entry in the capacity metrics table *$MetricsCapacityBlob* with the row key *'analytics'* shows the storage capacity consumed by the analytics logs.

This total capacity consumed by both user data and analytics logs (if enabled) can then be used to estimate the cost of storing data in the storage account. The same method can also be used for estimating storage costs for block and append blobs in general-purpose storage accounts.

### Transaction costs

The sum of *'TotalBillableRequests'*, across all entries for an API in the transaction metrics table indicates the total number of transactions for that particular API. *For example*, the total number of *'GetBlob'* transactions in a given period can be calculated by the sum of total billable requests for all entries with the row key *'user;GetBlob'*.

In order to estimate transaction costs for Blob storage accounts, you need to break down the transactions into three groups since they are priced differently.

* Write transactions such as *'PutBlob'*, *'PutBlock'*, *'PutBlockList'*, *'AppendBlock'*, *'ListBlobs'*, *'ListContainers'*, *'CreateContainer'*, *'SnapshotBlob'*, and *'CopyBlob'*.
* Delete transactions such as *'DeleteBlob'* and *'DeleteContainer'*.
* All other transactions.

In order to estimate transaction costs for general-purpose storage accounts, you need to aggregate all transactions irrespective of the operation/API.

### Data access and geo-replication data transfer costs

While storage analytics does not provide the amount of data read from and written to a storage account, it can be roughly estimated by looking at the transaction metrics table. The sum of *'TotalIngress'* across all entries for an API in the transaction metrics table indicates the total amount of ingress data in bytes for that particular API. Similarly the sum of *'TotalEgress'* indicates the total amount of egress data, in bytes.

In order to estimate the data access costs for Blob storage accounts, you need to break down the transactions into two groups.

* The amount of data retrieved from the storage account can be estimated by looking at the sum of *'TotalEgress'* for primarily the *'GetBlob'* and *'CopyBlob'* operations.

* The amount of data written to the storage account can be estimated by looking at the sum of *'TotalIngress'* for primarily the *'PutBlob'*, *'PutBlock'*, *'CopyBlob'* and *'AppendBlock'* operations.

The cost of geo-replication data transfer for Blob storage accounts can also be calculated by using the estimate for the amount of data written when using a GRS or RA-GRS storage account.

> [!NOTE]
> For a more detailed example about calculating the costs for using the hot or cool storage tier, take a look at the FAQ titled *'What are Hot and Cool access tiers and how should I determine which one to use?'* in the [Azure Storage Pricing Page](https://azure.microsoft.com/pricing/details/storage/).

## Migrating existing data

A Blob storage account is specialized for storing only block and append blobs. Existing general-purpose storage
accounts, which allow you to store tables, queues, files, and disks, as well as blobs, cannot be converted to Blob storage accounts. To use the storage tiers, you need to create new Blob storage accounts and migrate your existing data into the newly created accounts.

You can use the following methods to migrate existing data into Blob storage accounts from on-premises storage devices, from third-party cloud storage providers, or from your existing general-purpose storage accounts in Azure:

### AzCopy

AzCopy is a Windows command-line utility designed for high-performance copying of data to and from Azure Storage. You can use AzCopy to copy data into your Blob storage account from your existing general-purpose storage accounts, or to upload data from your on-premises storage devices into your Blob storage account.

For more details, see [Transfer data with the AzCopy Command-Line Utility](../common/storage-use-azcopy.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json).

### Data movement library

Azure Storage data movement library for .NET is based on the core data movement framework that powers AzCopy. The library is designed for high-performance, reliable, and easy data transfer operations similar to AzCopy. This allows you to take full benefits of the features provided by AzCopy in your application natively without having to deal with running and monitoring external instances of AzCopy.

For more details, see [Azure Storage Data Movement Library for .Net](https://github.com/Azure/azure-storage-net-data-movement)

### REST API or client library

You can create a custom application to migrate your data into a Blob storage account using one of the Azure client libraries or the Azure storage services REST API. Azure Storage provides rich client libraries for multiple languages and platforms like .NET, Java, C++, Node.JS, PHP, Ruby, and Python. The client libraries offer advanced capabilities such as retry logic, logging, and parallel uploads. You can also develop directly against the REST API, which can be called by any language that makes HTTP/HTTPS requests.

For more details, see [Get Started with Azure Blob storage](storage-dotnet-how-to-use-blobs.md).

> [!NOTE]
> Blobs encrypted using client-side encryption store encryption-related metadata stored with the blob. It is absolutely critical that any copy mechanism should ensure that the blob metadata, and especially the encryption-related metadata, is preserved. If you copy the blobs without this metadata, the blob content cannot be retrieved again. For more details regarding encryption-related metadata, see [Azure Storage Client-Side Encryption](../common/storage-client-side-encryption.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json).

## FAQ

1. **Are existing storage accounts still available?**

    Yes, existing storage accounts are still available and are unchanged in pricing or functionality.  They do not have the ability to choose a storage tier and will not have tiering capabilities in the future.

2. **Why and when should I start using Blob storage accounts?**

    Blob storage accounts are specialized for storing blobs and introduce new blob-centric features. Going forward, Blob storage accounts are the recommended way for storing blobs, as future capabilities such as hierarchical storage and tiering will be introduced based on this account type. However, it is up to you when you would like to migrate based on your business requirements.

3. **Can I convert my existing storage account to a Blob storage account?**

    No. Blob storage account is a different kind of storage account and you must create it new and migrate your data as explained previously.

4. **Can I store objects in both storage tiers in the same account?**

    Yes. The *'Access Tier'* attribute set at an account level is the default tier that applies to all objects in that account without an explicit set tier. However, blob-level tiering (preview) allows you to set the access tier on at the object level regardless of what the access tier setting on the account is. Blobs in any of the three storage tiers (hot, cool, or archive) may exist within the same account.

5. **Can I change the storage tier of my Blob storage account?**

    Yes, you can change the storage tier by setting the *'Access Tier'* attribute on the storage account. Changing the storage tier applies to all objects stored in the account that do not have an explicit tier set. Changing the storage tier from hot to cool incurs both write operations (per 10,000) and data write (per GB) charges (Blob storage accounts only), while changing from cool to hot incurs both read operations (per 10,000) and data retrieval (per GB) charges for reading all the data in the account.

6. **How frequently can I change the storage tier of my Blob storage account?**

    While we do not enforce a limitation on how frequently the storage tier can be changed, be aware that changing the storage tier from cool to hot can incur significant charges. Changing the storage tier frequently is not recommended.

7. **Do the blobs in the cool storage tier behave differently than the ones in the hot storage tier?**

    Blobs in the hot storage tier have the same latency as blobs in general-purpose storage accounts. Blobs in the cool storage tier have a similar latency (in milliseconds) as blobs in general-purpose storage accounts. Blobs in the archive storage tier have several hours of latency.

    Blobs in the cool storage tier have a slightly lower availability service level (SLA) than the blobs stored in the hot storage tier. For more details, see [SLA for storage](https://azure.microsoft.com/support/legal/sla/storage).

8. **Can I store page blobs and virtual machine disks in Blob storage accounts?**

    Blob storage accounts support only block and append blobs, and not page blobs. Azure virtual machine disks are backed by page blobs and as a result Blob storage accounts cannot be used to store virtual machine disks. However it is possible to store backups of the virtual machine disks as block blobs in a Blob storage account.

9. **Do I need to change my existing applications to use Blob storage accounts?**

    Blob storage accounts are 100% API consistent with general-purpose storage accounts for block and append blobs. As long as your application is using block blobs or append blobs, and you are using the 2014-02-14 version of the [Storage Services REST API](https://msdn.microsoft.com/library/azure/dd894041.aspx) or greater your application should work. If you are using an older version of the protocol, then you must update your application to use the new version so as to work seamlessly with both types of storage accounts. In general, we always recommend using the latest version regardless of which storage account type you use.

10. **Is there a change in user experience?**

    Blob storage accounts are very similar to general-purpose storage accounts for storing block and append blobs, and support all the key features of Azure Storage, including high durability and availability, scalability, performance, and security. Other than the features and restrictions specific to Blob storage accounts and its storage tiers that have been called out above, everything else remains the same.

## Next steps

### Evaluate Blob storage accounts

[Check availability of Blob storage accounts by region](https://azure.microsoft.com/regions/#services)

[Evaluate usage of your current storage accounts by enabling Azure Storage metrics](../common/storage-enable-and-view-metrics.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json)

[Check Blob storage pricing by region](https://azure.microsoft.com/pricing/details/storage/)

[Check data transfers pricing](https://azure.microsoft.com/pricing/details/data-transfers/)

### Start using Blob storage accounts

[Get Started with Azure Blob storage](storage-dotnet-how-to-use-blobs.md)

[Moving data to and from Azure Storage](../common/storage-moving-data.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json)

[Transfer data with the AzCopy Command-Line Utility](../common/storage-use-azcopy.md?toc=%2fazure%2fstorage%2fblobs%2ftoc.json)

[Browse and explore your storage accounts](http://storageexplorer.com/)
