---
title: iOS PhotoKit - 浏览照片和视频
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: ios-av
---

iOS 中的通过 PhotoKit 提供访问 "照片 App" 中的照片和视频，本文主要讲解使用 PhotoKit 浏览相册中的照片和视频、导出相册中的照片、导出相册中的视频、修改相册中的照片和新增相册中的照片。

![Camera Sea](/images/camera-sea.jpg)

完整代码：

* <em class="fab fa-github"></em> [PhotoLibraryViewController](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Library/PhotoLibraryViewController.swift)
* <em class="fab fa-github"></em> [LibraryPhotoCell](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Library/LibraryPhotoCell.swift)
* <em class="fab fa-github"></em> [LibraryVideoCell](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Library/LibraryVideoCell.swift)
* <em class="fab fa-github"></em> [LibraryAlbumCell](https://github.com/danjiang/DTCamera/blob/master/DTCamera/iOS/Library/LibraryAlbumCell.swift)

## 浏览相册中的照片和视频

第一步，获取相册使用权限：

{% highlight swift %}
PHPhotoLibrary.requestAuthorization { [weak self] status in
    guard let self = self else { return }
    switch status {
    case .authorized:
        self.imageManager = PHCachingImageManager()
        self.fetchAlbums()
        self.album = self.albums.first
        DispatchQueue.main.async { [weak self] in
            self?.updateAlbum()
            self?.tableView.reloadData()
        }
    default:
        DispatchQueue.main.async {
            DTMessageBar.error(message: "相册未授权", position: .bottom)
        }
    }
}
{% endhighlight %}

第二步，获取 Albums 和 Assets，先介绍一些核心类：

* PHImageManager 是操作相册的中心类，为了缓存和提升访问性能，这里使用子类 PHCachingImageManager。
* PHAssetCollection 是一组 Assets 的抽象，可以理解为相册。
* PHAsset 是一个 Asset 的抽象，可以是照片、视频或 Live Photo。

获取 PHAssetCollection：

{% highlight swift %}
var collections = PHAssetCollection.fetchAssetCollections(with: .smartAlbum,
                                                          subtype: .albumRegular,
                                                          options: nil)
var index = 0
while index < collections.count {
    let collection = collections.object(at: index)
    var subtypes: [PHAssetCollectionSubtype] = [.smartAlbumRecentlyAdded]
    if #available(iOS 9.0, *) {
        if mode.type != .video {
            subtypes.append(.smartAlbumScreenshots)
        }
    }
    if mode.type != .photo {
        subtypes.append(.smartAlbumVideos)
    }
    if subtypes.contains(collection.assetCollectionSubtype) {
        assets = fetchAssets(in: collection)
        album = Album(collection: collection, assets: assets)
        albums.append(album)
    }
    index += 1
}

collections = PHAssetCollection.fetchAssetCollections(with: .album, subtype: .albumRegular, options: nil)
index = 0
while index < collections.count {
    let collection = collections.object(at: index)
    assets = fetchAssets(in: collection)
    album = Album(collection: collection, assets: assets)
    albums.append(album)
    index += 1
}
{% endhighlight %}

获取 PHAsset，通过 NSPredicate 设置一些查询限定条件：

{% highlight swift %}
private func fetchAssets(in collection: PHAssetCollection? = nil) -> PHFetchResult<PHAsset> {
    if let collection = collection {
        let fetchOptions = PHFetchOptions()
        switch mode.type {
        case .photo:
            fetchOptions.predicate = NSPredicate(format: "mediaType = %d", PHAssetMediaType.image.rawValue)
        case .video:
            fetchOptions.predicate = NSPredicate(format: "mediaType = %d", PHAssetMediaType.video.rawValue)
        case .all:
            break
        }
        fetchOptions.sortDescriptors = [NSSortDescriptor(key: "creationDate", ascending: false)]
        return PHAsset.fetchAssets(in: collection, options: fetchOptions)
    } else {
        let fetchOptions = PHFetchOptions()
        fetchOptions.sortDescriptors = [NSSortDescriptor(key: "creationDate", ascending: false)]
        switch mode.type {
        case .photo:
            return PHAsset.fetchAssets(with: .image, options: fetchOptions)
        case .video:
            return PHAsset.fetchAssets(with: .video, options: fetchOptions)
        case .all:
            return PHAsset.fetchAssets(with: fetchOptions)
        }
    }
}
{% endhighlight %}

第三步，显示照片和视频的缩略图，PHAsset 包含的只是照片和视频的信息，缩略图要通过 PHImageManager 的 requestImage 方法异步获取：

{% highlight swift %}
func requestImage(for asset: PHAsset, targetSize: CGSize, contentMode: PHImageContentMode, options: PHImageRequestOptions?, resultHandler: @escaping (UIImage?, [AnyHashable : Any]?) -> Void) -> PHImageRequestID
{% endhighlight %}

PHImageRequestOptions 中的 PHImageRequestOptionsDeliveryMode 决定提供的照片质量和次数。

{% highlight swift %}
func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
    guard let assets = album?.assets else { return UICollectionViewCell() }
    let asset = assets.object(at: indexPath.row)
    if asset.mediaType == .image {
        let cell = collectionView.dequeueReusableCell(withReuseIdentifier: photoCellIdentifier, for: indexPath) as! LibraryPhotoCell
        cell.photoImageView.image = nil
        cell.representedAssetIdentifier = asset.localIdentifier
        if let imageManager = imageManager {
            let requestOptions = PHImageRequestOptions()
            requestOptions.isNetworkAccessAllowed = true
            imageManager.requestImage(for: asset,
                                      targetSize: thumbnailSize,
                                      contentMode: .aspectFill,
                                      options: requestOptions,
                                      resultHandler: { photo, _ in
                                        DispatchQueue.main.async {
                                            if cell.representedAssetIdentifier == asset.localIdentifier {
                                                cell.photoImageView.image = photo
                                            }
                                        }
            })
        }
        let index = selectedAssets.firstIndex { $0.localIdentifier == asset.localIdentifier }
        if let index = index {
            cell.setCheckNumber(index + 1)
            cell.toggleMask(isShow: false)
        } else {
            cell.setCheckNumber(nil)
            cell.toggleMask(isShow: selectedAssets.count >= mode.config.limitOfPhotos)
        }
        return cell
    } else {
        let cell = collectionView.dequeueReusableCell(withReuseIdentifier: videoCellIdentifier, for: indexPath) as! LibraryVideoCell
        cell.photoImageView.image = nil
        cell.representedAssetIdentifier = asset.localIdentifier
        if let imageManager = imageManager {
            let requestOptions = PHImageRequestOptions()
            requestOptions.isNetworkAccessAllowed = true
            imageManager.requestImage(for: asset,
                                      targetSize: thumbnailSize,
                                      contentMode: .aspectFill,
                                      options: requestOptions,
                                      resultHandler: { photo, _ in
                                        DispatchQueue.main.async {
                                            if cell.representedAssetIdentifier == asset.localIdentifier {
                                                cell.photoImageView.image = photo
                                            }
                                        }
            })
        }
        cell.setDuration(asset.duration)
        if asset.duration >= TimeInterval(mode.config.minDuration),
            asset.duration <= TimeInterval(mode.config.maxDuration) {
            cell.toggleMask(isShow: !selectedAssets.isEmpty)
        } else {
            cell.toggleMask(isShow: true)
        }
        return cell
    }
}
{% endhighlight %}

## 导出相册中的照片

和前面获取照片缩略图的代码一样，只是设置的照片质量不同：

{% highlight swift %}
private func fetchPhotos(completionBlock: @escaping (_ photos: [UIImage]?) -> Void) {
    guard let imageManager = imageManager else { return }
    let total = selectedAssets.count
    var current = 0
    var success = 0
    var photos = [UIImage](repeating: UIImage(), count: selectedAssets.count)
    let requestOptions = PHImageRequestOptions()
    requestOptions.deliveryMode = .highQualityFormat
    requestOptions.isNetworkAccessAllowed = true
    for (index, asset) in selectedAssets.enumerated() {
        imageManager.requestImage(for: asset,
                                  targetSize: PHImageManagerMaximumSize,
                                  contentMode: .aspectFit,
                                  options: requestOptions,
                                  resultHandler: { photo, _ in
                                    DispatchQueue.main.async {
                                        current += 1
                                        if let photo = photo {
                                            success += 1
                                            photos[index] = photo
                                        }
                                        if current == total {
                                            completionBlock(success == total ? photos : nil)
                                        }
                                    }
        })
    }
}
{% endhighlight %}

## 导出相册中的视频

PHImageManager 获取相册中的视频资源有如下几个方法：

* requestPlayerItem 获取可以直接播放的 AVPlayerItem。
* requestAVAsset 获取可以直接播放和编辑的 AVAsset。
* requestExportSession 获取可以导出到目录文件的 AVAssetExportSession。

这里使用 requestExportSession 获取 AVAssetExportSession，PHImageRequestOptions 和 PHVideoRequestOptions 都有关于从 iCloud 获取照片和视频的功能，isNetworkAccessAllowed 允许获取从 iCloud 获取照片和视频，progressHandler 获取从 iCloud 下载照片和视频的进度，PHImageManager cancelImageRequest 可以取消整个获取照片和视频的异步任务：

{% highlight swift %}
asset.duration >= TimeInterval(mode.config.minDuration),
asset.duration <= TimeInterval(mode.config.maxDuration) {
DTMessageHUD.hud()
guard let videoFile = MediaViewController.getMediaFileURL(name: "video", ext: "mp4"),
    let imageManager = imageManager else {
        DTMessageHUD.dismiss()
        DTMessageBar.error(message: "创建视频文件失败", position: .bottom)
        return
}
let requestOptions = PHVideoRequestOptions()
requestOptions.isNetworkAccessAllowed = true
let presets = AVAssetExportSession.allExportPresets()
var preset = presets.first ?? ""
if presets.contains(AVAssetExportPreset1280x720) {
    preset = AVAssetExportPreset1280x720
} else if presets.contains(AVAssetExportPresetMediumQuality) {
    preset = AVAssetExportPresetMediumQuality
}
imageManager.requestExportSession(forVideo: asset, options: requestOptions,
                                  exportPreset: preset) { [weak self] sess, _ in
                                    self?.exportVideo(videoFile, with: sess)
}
{% endhighlight %}

接着，AVAssetExportSession 通过 exportAsynchronously 异步导出相册中视频到沙盒中，cancelExport 取消导出，status 和 progress 获取状态和进度：

{% highlight swift %}
private func exportVideo(_ video: URL, with session: AVAssetExportSession?) {
    guard let session = session else {
        DispatchQueue.main.async {
            DTMessageHUD.dismiss()
            DTMessageBar.error(message: "获取视频导出会话失败", position: .bottom)
        }
        return
    }
    session.outputURL = video
    session.outputFileType = .mp4
    session.shouldOptimizeForNetworkUse = true
    session.exportAsynchronously { [weak self] in
        DispatchQueue.main.async { [weak self] in
            switch session.status {
            case .completed:
                DTMessageHUD.dismiss()
                self?.previewVideo(video)
            default:
                DTMessageHUD.dismiss()
                DTMessageBar.error(message: "视频文件导出失败", position: .bottom)
            }
        }
    }
}
{% endhighlight %}

## 修改相册中的照片

{% highlight swift %}
if asset.canPerform(.content) {
    asset.requestContentEditingInput(with: nil) { (contentEditingInput, _) in
        do {
            guard let contentEditingInput = contentEditingInput else { return }
            let contentEditingOutput = PHContentEditingOutput(contentEditingInput: contentEditingInput)
            let formatIdentifier = Bundle.main.bundleIdentifier ?? ""
            let cropping = "cropping".data(using: .utf8, allowLossyConversion: false)!
            contentEditingOutput.adjustmentData = PHAdjustmentData(formatIdentifier: formatIdentifier,
                                                                   formatVersion: "0.1",
                                                                   data: cropping)
            try croppedPhotoJPEG.write(to: contentEditingOutput.renderedContentURL,
                                       options: .atomicWrite)
            PHPhotoLibrary.shared().performChanges({
                let request = PHAssetChangeRequest(for: self.asset)
                request.contentEditingOutput = contentEditingOutput
            }, completionHandler: { (success, error) in
                if success {
                    print("modify success")
                } else {
                    print("modify fail \(error?.localizedDescription ?? "")")
                }
            })
        } catch {
            print("modify error \(error)")
        }
    }
}
{% endhighlight %}

## 新增相册中的照片

{% highlight swift %}
PHPhotoLibrary.shared().performChanges({
    let request = PHAssetChangeRequest.creationRequestForAsset(from: croppedPhoto)
}) { (success, _) in
    if success {
        print("save success")
    } else {
        print("save fail")
    }
}
{% endhighlight %}

