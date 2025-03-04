---
title: Sign up for Azure Video Indexer and upload your first video - Azure
description: Learn how to sign up and upload your first video using the Azure Video Indexer portal.
ms.topic: quickstart
ms.date: 08/24/2022
ms.author: juliako
ms.custom: mode-other
---

# Quickstart: How to sign up and upload your first video

[!INCLUDE [accounts](./includes/arm-accounts.md)]

[!INCLUDE [Gate notice](./includes/face-limited-access.md)]

You can access Azure Video Indexer capabilities in three ways:

* Azure Video Indexer portal: An easy-to-use solution that lets you evaluate the product, manage the account, and customize models (as described in this article).
* API integration: All of Azure Video Indexer's capabilities are available through a REST API, which lets you integrate the solution into your apps and infrastructure. To get started, see [Use Azure Video Indexer REST API](video-indexer-use-apis.md).
* Embeddable widget: Lets you embed the Azure Video Indexer insights, player, and editor experiences into your app. For more information, see [Embed visual widgets in your application](video-indexer-embed-widgets.md).

Once you start using Azure Video Indexer, all your stored data and uploaded content are encrypted at rest with a Microsoft managed key.

> [!NOTE]
> Review [planned Azure Video Indexer website authenticatication changes](./release-notes.md#planned-azure-video-indexer-website-authenticatication-changes).

This quickstart shows you how to sign in to the Azure Video Indexer [website](https://www.videoindexer.ai/) and how to upload your first video. 

[!INCLUDE [accounts](./includes/create-accounts-intro.md)]

## Sign up and upload a video

### Supported browsers

The following list shows the supported browsers that you can use for the Azure Video Indexer website and for your apps that embed the widgets. The list also shows the minimum supported browser version:

- Edge, version: 16
- Firefox, version: 54
- Chrome, version: 58
- Safari, version: 11
- Opera, version: 44
- Opera Mobile, version: 59
- Android Browser, version: 81
- Samsung Browser, version: 7
- Chrome for Android, version: 87
- Firefox for Android, version: 83

### Supported file formats for Azure Video Indexer

See the [input container/file formats](/azure/media-services/latest/encode-media-encoder-standard-formats-reference) article for a list of file formats that you can use with Azure Video Indexer.

### Upload 

1. Sign in on the [Azure Video Indexer](https://www.videoindexer.ai/) website.
1. To upload a video, press the **Upload** button or link.

    > [!NOTE]
    > The name of the video must be no greater than 80 characters.

    > [!div class="mx-imgBorder"]
    > :::image type="content" source="./media/video-indexer-get-started/video-indexer-upload.png" alt-text="Upload":::
1. Once your video has been uploaded, Azure Video Indexer starts indexing and analyzing the video. As a result a JSON output with insights is produced. 

    You see the progress.

    > [!div class="mx-imgBorder"]
    > :::image type="content" source="./media/video-indexer-get-started/progress.png" alt-text="Progress of the upload"::: 

    The produced JSON output contains `Insights` and `SummarizedInsights` elements. We highly recommend using `Insights` and not using `SummarizedInsights` (which is present for backward compatibility). 
    
1. Once Azure Video Indexer is done analyzing, you'll get an email with a link to your video and a short description of what was found in your video. For example: people, spoken and written words, topics, and named entities.
1. You can later find your video in the library list and perform different operations. For example: search, reindex, edit.

    > [!div class="mx-imgBorder"]
    > :::image type="content" source="./media/video-indexer-get-started/uploaded.png" alt-text="Uploaded the upload":::

After you upload and index a video, you can continue using [Azure Video Indexer website](video-indexer-view-edit.md) or [Azure Video Indexer Developer Portal](video-indexer-use-apis.md) to see the insights of the video (see [Examine the Azure Video Indexer output](video-indexer-output-json-v2.md)).

For more details, see [Upload and index videos](upload-index-videos.md).

To start using the APIs, see [use APIs](video-indexer-use-apis.md)

## Next steps

* For the API integration, see [Use Azure Video Indexer REST API](video-indexer-use-apis.md).
* To embed widgets, see [Embed visual widgets in your application](video-indexer-embed-widgets.md).
* Also, check out our [introduction lab](https://github.com/Azure-Samples/media-services-video-indexer/blob/master/IntroToVideoIndexer.md).

   At the end of the workshop, you'll have a good understanding of the kind of information that can be extracted from video and audio content, you'll be more    prepared to identify opportunities related to content intelligence, pitch video AI on Azure, and demo several scenarios on Azure Video Indexer.
