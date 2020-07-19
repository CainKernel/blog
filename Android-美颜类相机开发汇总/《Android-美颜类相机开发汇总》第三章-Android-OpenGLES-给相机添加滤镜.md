### 滤镜介绍
目前市面上的滤镜有很多，但整体归类也就几样，都是在fragment shader中进行处理。目前滤镜最常用的就是 lut滤镜以及调整RGB曲线的滤镜了。其他的类型变更大同小异。

### 动态滤镜的构建
为了实现动态下载的滤镜，我们接下来实现一套滤镜的json参数，主要包括滤镜类型、滤镜名称、vertex shader、fragment shader 文件、统一变量列表、与统一变量绑定的纹理图片、默认滤镜强度、是否带纹理宽高偏移量、音乐路径、音乐是否循环播放等参数。
json 以及各个字段的介绍如下：
```
{
    "filterList": [{
        "type": "filter",   // 表明滤镜类型，目前filter是只普通滤镜，后续还会加入其它类型的滤镜
        "name": "amaro",    // 滤镜名称
        "vertexShader": "", // vertex shader 文件名
        "fragmentShader": "fragment.glsl",  // fragment shader 文件名
        "uniformList":["blowoutTexture", "overlayTexture", "mapTexture"],   // 统一变量
        "uniformData": {  // 与统一变量绑定的纹理图片
            "blowoutTexture": "blowout.png",
            "overlayTexture": "overlay.png",
            "mapTexture": "map.png"
        },
        "strength": 1.0,        // 默认滤镜强度 0.0 ~ 1.0之间
        "texelOffset": 0,       // 是否需要支持宽高偏移值，即需要传递 1.0f/width, 1.0f/height到shader中
        "audioPath": "",        // 音乐路径
        "audioLooping": 1       // 是否循环播放音乐
    }]
}
```
有了json 之后，我们需要解码得到滤镜参数对象，解码如下：
```
/**
     * 解码滤镜数据
     * @param folderPath
     * @return
     */
    public static DynamicColor decodeFilterData(String folderPath)
            throws IOException, JSONException {

        File file = new File(folderPath, "json");
        String filterJson = FileUtils.convertToString(new FileInputStream(file));

        JSONObject jsonObject = new JSONObject(filterJson);
        DynamicColor dynamicColor = new DynamicColor();
        dynamicColor.unzipPath = folderPath;
        if (dynamicColor.filterList == null) {
            dynamicColor.filterList = new ArrayList<>();
        }

        JSONArray filterList = jsonObject.getJSONArray("filterList");
        for (int filterIndex = 0; filterIndex < filterList.length(); filterIndex++) {
            DynamicColorData filterData = new DynamicColorData();
            JSONObject jsonData = filterList.getJSONObject(filterIndex);
            String type = jsonData.getString("type");
            // TODO 目前滤镜只做普通的filter，其他复杂的滤镜类型后续在做处理
            if ("filter".equals(type)) {
                filterData.name = jsonData.getString("name");
                filterData.vertexShader = jsonData.getString("vertexShader");
                filterData.fragmentShader = jsonData.getString("fragmentShader");
                // 获取统一变量字段
                JSONArray uniformList = jsonData.getJSONArray("uniformList");
                for (int uniformIndex = 0; uniformIndex < uniformList.length(); uniformIndex++) {
                    String uniform = uniformList.getString(uniformIndex);
                    filterData.uniformList.add(uniform);
                }

                // 获取统一变量字段绑定的图片资源
                JSONObject uniformData = jsonData.getJSONObject("uniformData");
                if (uniformData != null) {
                    Iterator<String> dataIterator = uniformData.keys();
                    while (dataIterator.hasNext()) {
                        String key = dataIterator.next();
                        String value = uniformData.getString(key);
                        filterData.uniformDataList.add(new DynamicColorData.UniformData(key, value));
                    }
                }
                filterData.strength = (float) jsonData.getDouble("strength");
                filterData.texelOffset = (jsonData.getInt("texelOffset") == 1);
                filterData.audioPath = jsonData.getString("audioPath");
                filterData.audioLooping = (jsonData.getInt("audioLooping") == 1);
            }
            dynamicColor.filterList.add(filterData);
        }

        return dynamicColor;
    }
```

### 滤镜的实现
在解码得到滤镜参数之后，我们接下来实现动态滤镜渲染过程。为了方便构建滤镜，我们创建一个滤镜资源加载器，代码如下：
```
/**
 * 滤镜资源加载器
 */
public class DynamicColorLoader {

    private static final String TAG = "DynamicColorLoader";

    // 滤镜所在的文件夹
    private String mFolderPath;
    // 动态滤镜数据
    private DynamicColorData mColorData;
    // 资源索引加载器
    private ResourceDataCodec mResourceCodec;
    // 动态滤镜
    private final WeakReference<DynamicColorBaseFilter> mWeakFilter;
    // 统一变量列表
    private HashMap<String, Integer> mUniformHandleList = new HashMap<>();
    // 纹理列表
    private int[] mTextureList;

    // 句柄
    private int mTexelWidthOffsetHandle = OpenGLUtils.GL_NOT_INIT;
    private int mTexelHeightOffsetHandle = OpenGLUtils.GL_NOT_INIT;
    private int mStrengthHandle = OpenGLUtils.GL_NOT_INIT;
    private float mStrength = 1.0f;
    private float mTexelWidthOffset = 1.0f;
    private float mTexelHeightOffset = 1.0f;

    public DynamicColorLoader(DynamicColorBaseFilter filter, DynamicColorData colorData, String folderPath) {
        mWeakFilter = new WeakReference<>(filter);
        mFolderPath = folderPath.startsWith("file://") ? folderPath.substring("file://".length()) : folderPath;
        mColorData = colorData;
        mStrength = (colorData == null) ? 1.0f : colorData.strength;
        Pair pair = ResourceCodec.getResourceFile(mFolderPath);
        if (pair != null) {
            mResourceCodec = new ResourceDataCodec(mFolderPath + "/" + (String) pair.first, mFolderPath + "/" + pair.second);
        }
        if (mResourceCodec != null) {
            try {
                mResourceCodec.init();
            } catch (IOException e) {
                Log.e(TAG, "DynamicColorLoader: ", e);
                mResourceCodec = null;
            }
        }
        if (!TextUtils.isEmpty(mColorData.audioPath)) {
            if (mWeakFilter.get() != null) {
                mWeakFilter.get().setAudioPath(Uri.parse(mFolderPath + "/" + mColorData.audioPath));
                mWeakFilter.get().setLooping(mColorData.audioLooping);
            }
        }
        loadColorTexture();
    }

    /**
     * 加载纹理
     */
    private void loadColorTexture() {
        if (mColorData.uniformDataList == null || mColorData.uniformDataList.size() <= 0) {
            return;
        }
        mTextureList = new int[mColorData.uniformDataList.size()];
        for (int dataIndex = 0; dataIndex < mColorData.uniformDataList.size(); dataIndex++) {
            Bitmap bitmap = null;
            if (mResourceCodec != null) {
                bitmap = mResourceCodec.loadBitmap(mColorData.uniformDataList.get(dataIndex).value);
            }
            if (bitmap == null) {
                bitmap = BitmapUtils.getBitmapFromFile(mFolderPath + "/" + String.format(mColorData.uniformDataList.get(dataIndex).value));
            }
            if (bitmap != null) {
                mTextureList[dataIndex] = OpenGLUtils.createTexture(bitmap);
                bitmap.recycle();
            } else {
                mTextureList[dataIndex] = OpenGLUtils.GL_NOT_TEXTURE;
            }
        }
    }


    /**
     * 绑定统一变量句柄
     * @param programHandle
     */
    public void onBindUniformHandle(int programHandle) {
        if (programHandle == OpenGLUtils.GL_NOT_INIT || mColorData == null) {
            return;
        }
        mStrengthHandle = GLES30.glGetUniformLocation(programHandle, "strength");
        if (mColorData.texelOffset) {
            mTexelWidthOffsetHandle = GLES30.glGetUniformLocation(programHandle, "texelWidthOffset");
            mTexelHeightOffsetHandle = GLES30.glGetUniformLocation(programHandle, "texelHeightOffset");
        } else {
            mTexelWidthOffsetHandle = OpenGLUtils.GL_NOT_INIT;
            mTexelHeightOffsetHandle = OpenGLUtils.GL_NOT_INIT;
        }
        for (int uniformIndex = 0; uniformIndex < mColorData.uniformList.size(); uniformIndex++) {
            String uniformString = mColorData.uniformList.get(uniformIndex);
            int handle = GLES30.glGetUniformLocation(programHandle, uniformString);
            mUniformHandleList.put(uniformString, handle);
        }
    }

    /**
     * 输入纹理大小
     * @param width
     * @param height
     */
    public void onInputSizeChange(int width, int height) {
        mTexelWidthOffset = 1.0f / width;
        mTexelHeightOffset = 1.0f / height;
    }

    /**
     * 绑定滤镜纹理，只需要绑定一次就行，不用重复绑定，减少开销
     */
    public void onDrawFrameBegin() {
        if (mStrengthHandle != OpenGLUtils.GL_NOT_INIT) {
            GLES30.glUniform1f(mStrengthHandle, mStrength);
        }
        if (mTexelWidthOffsetHandle != OpenGLUtils.GL_NOT_INIT) {
            GLES30.glUniform1f(mTexelWidthOffsetHandle, mTexelWidthOffset);
        }
        if (mTexelHeightOffsetHandle != OpenGLUtils.GL_NOT_INIT) {
            GLES30.glUniform1f(mTexelHeightOffsetHandle, mTexelHeightOffset);
        }

        if (mTextureList == null || mColorData == null) {
            return;
        }
        // 逐个绑定纹理
        for (int dataIndex = 0; dataIndex < mColorData.uniformDataList.size(); dataIndex++) {
            for (int uniformIndex = 0; uniformIndex < mUniformHandleList.size(); uniformIndex++) {
                // 如果统一变量存在，则直接绑定纹理
                Integer handle = mUniformHandleList.get(mColorData.uniformDataList.get(dataIndex).uniform);
                if (handle != null && mTextureList[dataIndex] != OpenGLUtils.GL_NOT_TEXTURE) {
                    OpenGLUtils.bindTexture(handle, mTextureList[dataIndex], dataIndex + 1);
                }
            }
        }
    }

    /**
     * 释放资源
     */
    public void release() {
        if (mTextureList != null && mTextureList.length > 0) {
            GLES30.glDeleteTextures(mTextureList.length, mTextureList, 0);
            mTextureList = null;
        }
        if (mWeakFilter.get() != null) {
            mWeakFilter.clear();
        }
    }

    /**
     * 设置强度
     * @param strength
     */
    public void setStrength(float strength) {
        mStrength = strength;
    }

}
```
然后我们构建一个DynamicColorFilter的基类，方便后续添加其他类型的滤镜，代码如下：
```
public class DynamicColorBaseFilter extends GLImageAudioFilter {

    // 颜色滤镜参数
    protected DynamicColorData mDynamicColorData;
    protected DynamicColorLoader mDynamicColorLoader;

    public DynamicColorBaseFilter(Context context, DynamicColorData dynamicColorData, String unzipPath) {
        super(context, (dynamicColorData == null || TextUtils.isEmpty(dynamicColorData.vertexShader)) ? VERTEX_SHADER
                        : getShaderString(context, unzipPath, dynamicColorData.vertexShader),
                (dynamicColorData == null || TextUtils.isEmpty(dynamicColorData.fragmentShader)) ? FRAGMENT_SHADER_2D
                        : getShaderString(context, unzipPath, dynamicColorData.fragmentShader));
        mDynamicColorData = dynamicColorData;
        mDynamicColorLoader = new DynamicColorLoader(this, mDynamicColorData, unzipPath);
        mDynamicColorLoader.onBindUniformHandle(mProgramHandle);
    }

    @Override
    public void onInputSizeChanged(int width, int height) {
        super.onInputSizeChanged(width, height);
        if (mDynamicColorLoader != null) {
            mDynamicColorLoader.onInputSizeChange(width, height);
        }
    }

    @Override
    public void onDrawFrameBegin() {
        super.onDrawFrameBegin();
        if (mDynamicColorLoader != null) {
            mDynamicColorLoader.onDrawFrameBegin();
        }
    }

    @Override
    public void release() {
        super.release();
        if (mDynamicColorLoader != null) {
            mDynamicColorLoader.release();
        }
    }

    /**
     * 设置强度，调节滤镜的轻重程度
     * @param strength
     */
    public void setStrength(float strength) {
        if (mDynamicColorLoader != null) {
            mDynamicColorLoader.setStrength(strength);
        }
    }

    /**
     * 根据解压路径和shader名称读取shader的字符串内容
     * @param unzipPath
     * @param shaderName
     * @return
     */
    protected static String getShaderString(Context context, String unzipPath, String shaderName) {
        if (TextUtils.isEmpty(unzipPath) || TextUtils.isEmpty(shaderName)) {
            throw new IllegalArgumentException("shader is empty!");
        }
        String path = unzipPath + "/" + shaderName;
        if (path.startsWith("assets://")) {
            return OpenGLUtils.getShaderFromAssets(context, path.substring("assets://".length()));
        } else if (path.startsWith("file://")) {
            return OpenGLUtils.getShaderFromFile(path.substring("file://".length()));
        }
        return OpenGLUtils.getShaderFromFile(path);
    }

}
```
接下来我们构建动态滤镜组，因为动态滤镜有可能有多个滤镜组合而成。代码如下：
```
public class GLImageDynamicColorFilter extends GLImageGroupFilter {

    public GLImageDynamicColorFilter(Context context, DynamicColor dynamicColor) {
        super(context);
        // 判断数据是否存在
        if (dynamicColor == null || dynamicColor.filterList == null
                || TextUtils.isEmpty(dynamicColor.unzipPath)) {
            return;
        }
        // 添加滤镜
        for (int i = 0; i < dynamicColor.filterList.size(); i++) {
            mFilters.add(new DynamicColorFilter(context, dynamicColor.filterList.get(i), dynamicColor.unzipPath));
        }
    }

    /**
     * 设置滤镜强度
     * @param strength
     */
    public void setStrength(float strength) {
        for (int i = 0; i < mFilters.size(); i++) {
            if (mFilters.get(i) != null && mFilters.get(i) instanceof DynamicColorBaseFilter) {
                ((DynamicColorBaseFilter) mFilters.get(i)).setStrength(strength);
            }
        }
    }
}
```
### 总结
基本的动态滤镜实现起来比较简单，总的来说就是简单的json参数、shader、统一变量和纹理绑定需要做成动态构建的过程而已。
效果如下：
![动态滤镜效果](https://upload-images.jianshu.io/upload_images/2103804-5ebaf557239c7e25.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
该效果是通过解压asset目录下的压缩包资源来实现的。你只需要提供包含shader 、纹理资源、以及json的压缩包即可更改滤镜。

详细实现过程，可参考本人的开源项目：
[CainCamera](https://github.com/CainKernel/CainCamera)
