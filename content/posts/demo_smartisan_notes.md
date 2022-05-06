---
title: 临摹锤子便签--SmartisanNotes
date: '2016-10-23'
spoiler: 关于锤子便签的一点尝试
---

因为前一段时间一直在做一个应用，其中有个类似生成长微博的需求，当时第一反应就是锤子便签，毕竟我还算是老罗的粉丝，对锤子便签的生成图片功能一直情有独钟，之前的想法是锤子把这个功能做成了API，点击生成图片的时候把文字和图片数据传上去，然后下载生成的图片，为什么会这么想呢，因为锤子便签是支持多平台的，Android，iOS以及Web端都支持生成图片的功能，最方便的方法就是利用API生成，但是最终的结果跟我想象的并不一样，锤子便签并没有调用API，这也让我想要偷懒直接使用他们的API生成图片的幻想破灭了，自己动手丰衣足食，于是有了SmartisanNotes这个项目。

<!-- more -->

整个项目下来花了差不多7、8个小时吧，总体感觉还不错，学到了很多东西，关于Canvas的，关于Path的，还有关于文件保存的，总之收获还是不小的，以后会尽量抽一些完整的时间做一些小项目，力求精致，不求大而全，认认真真写好每一行代码。

### 拍照保存

```
			ContentValues values = new ContentValues();
            values.put(MediaStore.Images.Media.DATE_ADDED, System.currentTimeMillis());
            values.put(MediaStore.Images.Media.DATE_TAKEN, System.currentTimeMillis());
            values.put(MediaStore.Images.Media.DATE_MODIFIED, System.currentTimeMillis());

            imageFileUri = getActivity().getContentResolver()
                    .insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values);
```
首先new一个 `ContentValues` 对象，设置这个对象的三个属性：`DATE_ADDED`, `DATE_TAKEN`, `DATE_MODIFIED`, 依次对象照片文件的新增日期，拍照日期和修改日期，然后在通过 `ContentResolver` 通过 insert 方法插入到 contentProvider 中，这样我们就获得了一个imageFileUri，然后通过一定的操作就可以通过Uri得到保存图片的具体路径。

### 文件保存

```
File resultDir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES)        resultDir.mkdirs();
File result = new File(resultDir, System.currentTimeMillis() + ".jpg");
```
通过 `getExternalStoragePublicDirectory()` 方法获得文件存储的具体文件夹，系统会根据传入的不同参数返回不同的文件夹，然后在文件夹中创建需要保存的文件，然后保存就OK了。
```
MediaScannerConnection.scanFile(mContext, new String[] { file.toString() }, null,
                new MediaScannerConnection.OnScanCompletedListener() {
                    @Override
                    public void onScanCompleted(String path, Uri uri) {
                        Log.i("ExternalStorage", "Scanned " + path + ":");
                        Log.i("ExternalStorage", "-> uri=" + uri);
                    }
                });
```
如果保存的是图片，可能还需要加上这段代码，目的是通过 `MediaScannerConnection` 传递一个新建的或者新下载的多媒体文件到 `Media Scanner Service`， 这个 Service 会把文件信息读取然后添加到 `Content Provider`中，而且还可以通过一个接口返回在`Content Provider` 中保存的Uri。