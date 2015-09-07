>本教程采用阿里dexposed开源库实现。
https://github.com/alibaba/dexposed

##主APP实现：

###主程序Application onCreate方法中初始化dexposed
		DexposedBridge.canDexposed(context);
### Patch apk下载及修复：
1. 为保证修复patch的及时性，使用push推送patch，客户端收到消息后立即完成patch的下载及修复；
2. 客户端版本管理模块在程序入口Activity中检测是否有需要修复的patch；
3. 下载完patch apk到程序私有目录，即/data/data/packageName/files目录，同时可在xml中保存patch apk本地存储路径、方便下载启动app时加载补丁patch。

```java
public class HotPatchManager {
public static boolean canDexposed = false;
private static final String SP_KEY_HOT_PATCH = "hot_patch_path";

	/**
	 * init hotPatch library.
	 *
	 * @param Context
	 */
	public static void init(Context context) {
	    // aop init.
	    canDexposed = DexposedBridge.canDexposed(context);
	    if (canDexposed) {
	        List<String> list = getHotPatchPaths(context);
	        if (list != null && list.size() > 0) {
	            for (String path : list) {
	                runPatchApk(context, path);
	            }
	        }
	    } else {
	        if (LogUtils.DEBUG) {
	            LogUtils.d("==========your device not support dexposed aop.==========");
	        }
	    }
	}

	/**
	 * /data/data/package/files
	 *
	 * @param context
	 * @param apkPath
	 */
	public static void runPatchApk(Context context, String apkPath) {
	    if (Build.VERSION.SDK_INT >= 21 || !canDexposed) {
	        LogUtils.d("This device doesn't support dexposed.");
	        return;
	    }
	    if (!pathIsValid(context, apkPath)) {
	        return;
	    }
	    try {
	        PatchResult result = PatchMain.load(context, apkPath, null);
	        if (result.isSuccess()) {
	            LogUtils.d("hotPath load apk success.");
	        } else {
	            LogUtils.e("hotPath load apk error.", result.getErrorInfo());
	            result.getThrowbale().printStackTrace();
	        }
	    } catch (Exception e) {
	        e.printStackTrace();
	    }
	}

	/**
	 * download hotPatch and auto mege.
	 *
	 * @param context
	 */
	public static void downloadHotPatch(final Context context, String downloadUrl) {
	    if (TextUtils.isEmpty(downloadUrl)) {
	        LogUtils.d("downloadUrl is null.");
	        return;
	    }
	    DownloadInfo downloadInfo = new DownloadInfo();
	    downloadInfo.setDownloadUrl(downloadUrl);
	    String fileName = downloadUrl.substring(downloadUrl.lastIndexOf("/") + 1);
	    String fileSavePath = new File(context.getFilesDir(), fileName).getAbsolutePath();
	    downloadInfo.setFileSavePath(fileSavePath);
	    downloadInfo.setDaoCallback(
	            new Task.Callback() {
	                @Override
	                public void onSuccess(DownloadInfo downloadInfo) {
	                    LogUtils.d("runPatchApk begin.", downloadInfo.getFileSavePath());
	                    runPatchApk(context, downloadInfo.getFileSavePath());
	                    appendHotPatchPath(context, downloadInfo.getFileSavePath());
	                    LogUtils.d("runPatchApk end.", downloadInfo.getFileSavePath());
	                }

	                @Override
	                public void onStart(DownloadInfo downloadInfo) {
	                }

	                @Override
	                public void onFailure(DownloadInfo downloadInfo) {
	                }

	                @Override
	                public boolean onLoading(long total, long current) {
	                    return true;
	                }

	                @Override
	                public void onCancelled(DownloadInfo downloadInfo) {
	                }
	            }
	    );
	    DownloadManager dm = DownloadService.getDownloadManager(context, DownloadService.ACTION);
	    dm.addDownloadTask(downloadInfo);
	}

	public static void clearHotPatchFiles(Context context) {
	    List<String> list = getHotPatchPaths(context);
	    if (list != null && list.size() > 0) {
	        for (String path : list) {
	            FileUtils.delFile(path);
	        }
	    }
	}

	public static boolean pathIsValid(Context context, String apkPath) {
	    if (TextUtils.isEmpty(apkPath)) {
	        LogUtils.d("apkPath is null.");
	        return false;
	    }
	    String parentDir = String.format("/data/data/%s/files", context.getPackageName());
	    File apkFile = new File(apkPath);
	    if (!parentDir.equals(apkFile.getParent())) {
	        LogUtils.d("apkPath is error.", apkPath);
	        return false;
	    }
	    if (!apkFile.exists()){
	        LogUtils.d("apkPath is not exist.", apkPath);
	        return false;
	    }
	    return true;
	}

	public static List<String> getHotPatchPaths(Context context) {
	    List<String> list = null;
	    SP sp = SP.getInstance(context);
	    String paths = sp.getString(SP_KEY_HOT_PATCH, null);
	    if (!TextUtils.isEmpty(paths)) {
	        if (paths.indexOf(",") != -1) {
	            String[] pathArr = paths.split(",");
	            if (pathArr != null && pathArr.length > 0) {
	                list = Arrays.asList(paths);
	            }
	        } else {
	            list = new ArrayList<String>();
	            list.add(paths);
	        }
	    }
	    return list;
	}

	public static void appendHotPatchPath(Context context, String apkPath) {
	    if (!pathIsValid(context, apkPath)) {
	        return;
	    }

	    SP sp = SP.getInstance(context);
	    String paths = sp.getString(SP_KEY_HOT_PATCH, null);
	    if (!TextUtils.isEmpty(paths)) {
	        String allPath = new StringBuilder(apkPath).append(",").append(apkPath).toString();
	        sp.commit(SP_KEY_HOT_PATCH, allPath);
	    } else {
	        sp.commit(SP_KEY_HOT_PATCH, apkPath);
	    }
	}

	public static void clearHotPatchPaths(Context context) {
	    SP sp = SP.getInstance(context);
	    sp.commit(SP_KEY_HOT_PATCH, "");
	}

}
```
		
##Patch Apk部分：
>dexpose支持方法粒度的patch，可以实现整个方法的替换或方法前、后执行修复代码。
以下实例为方法替换实例，其它只需实现相应的回调接口即可。

###方法替换实例：
1. 新建Android工程，引入patchloader.jar、dexposedbridge.jar；
2. 创建Patch修复类实现IPatch接口；

		```java
		public class HotPatch implements IPatch {

		@Override
		public void handlePatch(final PatchParam arg0) throws Throwable {    	
	    	Class<?> cls = null;
			try {
				cls= arg0.context.getClassLoader()
						.loadClass("com.zaozuo.app.MainActivity");
			} catch (ClassNotFoundException e) {
				e.printStackTrace();
				return;
			}     	
	     	DexposedBridge.findAndHookMethod(cls, "bindData",
					new XC_MethodReplacement() {
				@Override
				protected Object replaceHookedMethod(MethodHookParam param) throws Throwable {
					Activity mainActivity = (Activity) param.thisObject;
					Toast.makeText(mainActivity, "test show hotPatch.",Toast.LENGTH_LONG).show();
					return null;                 
				}
			});
		}

		}
		```
	
3. 打包patch apk，上传到服务器并通知客户端下载。

### Patch Apk安全性：
1. 打包apk必须使用主app签名文件签名；
2. 主app对加载的patch apk做签名和无篡改校验：


![手机扫码快速访问](http://upload-images.jianshu.io/upload_images/711578-70dc9b6a8049b976.png?imageMogr2/auto-orient/strip|imageView2/2/w/1240)