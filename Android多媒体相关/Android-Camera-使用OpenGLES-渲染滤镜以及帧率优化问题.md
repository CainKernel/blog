说到滤镜问题，市面上所有美颜类的相机都存在各式各样的滤镜。那么我们怎么实现滤镜呢？我们首先想到，是否有相关开源项目可以参考的。iOS 下有比较著名的GPUImage是用来做滤镜渲染的，Android下面也有类似的项目。其中，美颜类开源相机比较出名的是程序员杠把子(CSDN博客：http://my.csdn.net/oShunz)的MagicCamera(github地址：https://github.com/wuhaoyu1990/MagicCamera)了。但是我发现MagicCamera的帧率实在是太低了，尤其是在使用了滤镜之后，在红米Note2上，发现跑起来的帧率只有13帧，这还是只做了磨皮和滤镜的情况下，如下图的log所示：
![MagicCamera预览帧率](http://upload-images.jianshu.io/upload_images/2103804-bb57afc3377e8769.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

相比市面上的美颜类相机来说，这样的帧率太低了。因为对商业相机来说，不仅仅需要做磨皮，还要做人脸关键点检测，检测完成后需要做瘦脸大眼等比较耗时的渲染处理，这么低的帧率肯定是不能满足需要的。为此，本人开发了自己的相机：CainCamera(github地址：[CainCamera](https://github.com/CainKernel/CainCamera))。在开发CainCamera相机过程中，在写滤镜的过程中，很大程度上参考了MagicCamera，对此，表示非常感谢程序员杠把子所做的努力和提供了这么棒的开源相机和滤镜。

废话不多说，下面开始进入正题。本人开发的相机帧率究竟如何呢？同样是做了磨皮和滤镜渲染，CainCamera在红米Note2上做和MagicCamera同样的磨皮和滤镜时，实时预览的帧率稳定在19帧以上，如下图所示：
![CainCamera预览帧率](http://upload-images.jianshu.io/upload_images/2103804-244e9eaefca7cc9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
关于市面上主流的美颜类相机的帧率表现，可以参考本人的文章：[Android预览实时渲染的帧率优化相关](http://www.jianshu.com/p/7187fe10261d)，里面有跟市面上主流美颜类相机做过比较

首先是所有滤镜的基类构建问题。基类写得好，不仅可以节省具体滤镜的代码量，甚至能够提高渲染效率。下面来看看本人写的滤镜基类是如何构成的：
CainCamera的滤镜基类由BaseImageFilter 和 BaseImageFilterGroup构成的。BaseImageFilter是所有滤镜和滤镜组的基类，BaseImageFilterGroup则是所有滤镜组的基类，继承于BaseImageFilter。滤镜组是用于管理渲染多个滤镜的渲染。下面我们来看看两个基类的写法：
首先是BaseImageFilter， BaseImageFilter的核心方法如下:
```
/**
     * 绘制Frame
     * @param textureId
     */
    public boolean drawFrame(int textureId) {
        return drawFrame(textureId, mVertexArray, mTexCoordArray);
    }

    /**
     * 绘制Frame
     * @param textureId
     * @param vertexBuffer
     * @param textureBuffer
     */
    public boolean drawFrame(int textureId, FloatBuffer vertexBuffer,
                          FloatBuffer textureBuffer) {
        if (textureId == GlUtil.GL_NOT_INIT) {
            return false;
        }
        GLES30.glUseProgram(mProgramHandle);
        runPendingOnDrawTasks();

        vertexBuffer.position(0);
        GLES30.glVertexAttribPointer(maPositionLoc, mCoordsPerVertex,
                GLES30.GL_FLOAT, false, 0, vertexBuffer);
        GLES30.glEnableVertexAttribArray(maPositionLoc);

        textureBuffer.position(0);
        GLES30.glVertexAttribPointer(maTextureCoordLoc, 2,
                GLES30.GL_FLOAT, false, 0, textureBuffer);
        GLES30.glEnableVertexAttribArray(maTextureCoordLoc);

        GLES30.glUniformMatrix4fv(muMVPMatrixLoc, 1, false, mMVPMatrix, 0);
        GLES30.glUniformMatrix4fv(mTexMatrixLoc, 1, false, mTexMatrix, 0);
        GLES30.glActiveTexture(GLES30.GL_TEXTURE0);
        GLES30.glBindTexture(getTextureType(), textureId);
        GLES30.glUniform1i(mInputTextureLoc, 0);
        onDrawArraysBegin();
        GLES30.glDrawArrays(GLES30.GL_TRIANGLE_STRIP, 0, mVertexCount);
        onDrawArraysAfter();
        GLES30.glDisableVertexAttribArray(maPositionLoc);
        GLES30.glDisableVertexAttribArray(maTextureCoordLoc);
        GLES30.glBindTexture(getTextureType(), 0);
        GLES30.glUseProgram(0);
        return true;
    }

    /**
     * 绘制到FBO
     * @param textureId
     * @return FBO绑定的Texture
     */
    public int drawFrameBuffer(int textureId) {
        return drawFrameBuffer(textureId, mVertexArray, mTexCoordArray);
    }

    /**
     * 绘制到FBO
     * @param textureId
     * @param vertexBuffer
     * @param textureBuffer
     * @return FBO绑定的Texture
     */
    public int drawFrameBuffer(int textureId, FloatBuffer vertexBuffer, FloatBuffer textureBuffer) {
        if (mFramebuffers == null) {
            return GlUtil.GL_NOT_INIT;
        }
        runPendingOnDrawTasks();
        GLES30.glViewport(0, 0, mFrameWidth, mFrameHeight);
        GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, mFramebuffers[0]);
        GLES30.glUseProgram(mProgramHandle);
        vertexBuffer.position(0);
        GLES30.glVertexAttribPointer(maPositionLoc, mCoordsPerVertex,
                GLES30.GL_FLOAT, false, 0, vertexBuffer);
        GLES30.glEnableVertexAttribArray(maPositionLoc);

        textureBuffer.position(0);
        GLES30.glVertexAttribPointer(maTextureCoordLoc, 2,
                GLES30.GL_FLOAT, false, 0, textureBuffer);
        GLES30.glEnableVertexAttribArray(maTextureCoordLoc);

        GLES30.glUniformMatrix4fv(muMVPMatrixLoc, 1, false, mMVPMatrix, 0);
        GLES30.glActiveTexture(GLES30.GL_TEXTURE0);
        GLES30.glBindTexture(getTextureType(), textureId);
        GLES30.glUniform1i(mInputTextureLoc, 0);
        onDrawArraysBegin();
        GLES30.glDrawArrays(GLES30.GL_TRIANGLE_STRIP, 0, mVertexCount);
        onDrawArraysAfter();
        GLES30.glDisableVertexAttribArray(maPositionLoc);
        GLES30.glDisableVertexAttribArray(maTextureCoordLoc);
        GLES30.glBindTexture(getTextureType(), 0);
        GLES30.glUseProgram(0);
        GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, 0);
        GLES30.glViewport(0, 0, mDisplayWidth, mDisplayHeight);
        return mFramebufferTextures[0];
    }
```
基类的核心方法就两个：drawFrame 和 drawFrameBuffer。drawFrame方法直接绘制渲染输入。drawFrameBuffer方法，则将texture渲染到FBO中，该方法用于多重滤镜的渲染上。2017年12月08日更新： 这里的drawFrame和drawFrameBuffer方法都做了更新，因为本人实现了多段视频录制的功能之后，发现这里存在一个小Bug，就是处于录制视频状态下，切换滤镜会产生一帧的黑屏现象，由于本人使用OpenGLES 的 multi thread multi rendering context做录制操作，所以在这里存在一个线程同步的问题，多线程环境下切换滤镜，需要考虑FBO是否正确绑定texture了。不过这里也很好改动，如果没有绑定，则返回一个标志，让FBO绑定跳过就好，因为此时并没有当前滤镜层的渲染操作，否则会将FBO绑定到一个空的Texture 中，导致录制时切换滤镜瞬间存在黑屏的现象，该问题已修复。

接下来我们来看看BaseImageFilterGroup的核心：
```
   @Override
    public void onInputSizeChanged(int width, int height) {
        super.onInputSizeChanged(width, height);
        if (mFilters.size() <= 0) {
            return;
        }
        int size = mFilters.size();
        for (int i = 0; i < size; i++) {
            mFilters.get(i).onInputSizeChanged(width, height);
        }
        // 先销毁原来的Framebuffers
        if(mFramebuffers != null && (mImageWidth != width
                || mImageHeight != height || mFramebuffers.length != size-1)) {
            destroyFramebuffer();
            mImageWidth = width;
            mImageWidth = height;
        }
        initFramebuffer(width, height);
    }

    @Override
    public void onDisplayChanged(int width, int height) {
        super.onDisplayChanged(width, height);
        // 更新显示的的视图大小
        if (mFilters.size() <= 0) {
            return;
        }
        int size = mFilters.size();
        for (int i = 0; i < size; i++) {
            mFilters.get(i).onDisplayChanged(width, height);
        }
    }

    @Override
    public boolean drawFrame(int textureId) {
        if (mFramebuffers == null || mFrameBufferTextures == null || mFilters.size() <= 0) {
            return false;
        }
        int size = mFilters.size();
        mCurrentTextureId = textureId;
        for (int i = 0; i < size; i++) {
            BaseImageFilter filter = mFilters.get(i);
            if (i < size - 1) {
                GLES30.glViewport(0, 0, mImageWidth, mImageHeight);
                GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, mFramebuffers[i]);
                GLES30.glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
                if (filter.drawFrame(mCurrentTextureId)) {
                    mCurrentTextureId = mFrameBufferTextures[i];
                }
                GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, 0);
            } else {
                GLES30.glViewport(0, 0, mDisplayWidth, mDisplayHeight);
                filter.drawFrame(mCurrentTextureId);
            }
        }
        return true;
    }

    @Override
    public boolean drawFrame(int textureId, FloatBuffer vertexBuffer, FloatBuffer textureBuffer) {
        if (mFramebuffers == null || mFrameBufferTextures == null || mFilters.size() <= 0) {
            return false;
        }
        int size = mFilters.size();
        mCurrentTextureId = textureId;
        for (int i = 0; i < size; i++) {
            BaseImageFilter filter = mFilters.get(i);
            if (i < size - 1) {
                GLES30.glViewport(0, 0, mImageWidth, mImageHeight);
                GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, mFramebuffers[i]);
                GLES30.glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
                filter.drawFrame(mCurrentTextureId, vertexBuffer, textureBuffer);
                GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, 0);
                mCurrentTextureId = mFrameBufferTextures[i];
            } else {
                GLES30.glViewport(0, 0, mDisplayWidth, mDisplayHeight);
                filter.drawFrame(mCurrentTextureId, vertexBuffer, textureBuffer);
            }
        }
        return true;
    }

    @Override
    public int drawFrameBuffer(int textureId) {
        if (mFramebuffers == null || mFrameBufferTextures == null || mFilters.size() <= 0) {
            return textureId;
        }
        int size = mFilters.size();
        mCurrentTextureId = textureId;
        GLES30.glViewport(0, 0, mImageWidth, mImageHeight);
        for (int i = 0; i < size; i++) {
            GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, mFramebuffers[i]);
            GLES30.glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
            if (mFilters.get(i).drawFrame(mCurrentTextureId)) {
                mCurrentTextureId = mFrameBufferTextures[i];
            }
            GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, 0);
        }
        return mCurrentTextureId;
    }

    @Override
    public int drawFrameBuffer(int textureId, FloatBuffer vertexBuffer, FloatBuffer textureBuffer) {
        if (mFramebuffers == null || mFrameBufferTextures == null || mFilters.size() <= 0) {
            return textureId;
        }
        int size = mFilters.size();
        mCurrentTextureId = textureId;
        GLES30.glViewport(0, 0, mImageWidth, mImageHeight);
        for (int i = 0; i < size; i++) {
            GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, mFramebuffers[i]);
            GLES30.glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
            if (mFilters.get(i).drawFrame(mCurrentTextureId, vertexBuffer, textureBuffer)) {
                mCurrentTextureId = mFrameBufferTextures[i];
            }
            GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, 0);
        }
        return mCurrentTextureId;
    }

    @Override
    public void release() {
        if (mFilters != null) {
            for (BaseImageFilter filter : mFilters) {
                filter.release();
            }
            mFilters.clear();
        }
        destroyFramebuffer();
    }

    /**
     * 初始化framebuffer，这里在调用drawFrame时，会多一个FBO，这里为了方便后面录制视频缩放处理
     */
    public void initFramebuffer(int width, int height) {
        int size = mFilters.size();
        // 创建Framebuffers 和 Textures
        if (mFramebuffers == null) {
            mFramebuffers = new int[size];
            mFrameBufferTextures = new int[size];
            createFramebuffer(0, size);
        }
    }

    /**
     * 创建Framebuffer
     * @param start
     * @param size
     */
    private void createFramebuffer(int start, int size) {
        for (int i = start; i < size; i++) {
            GLES30.glGenFramebuffers(1, mFramebuffers, i);

            GLES30.glGenTextures(1, mFrameBufferTextures, i);
            GLES30.glBindTexture(GLES30.GL_TEXTURE_2D, mFrameBufferTextures[i]);

            GLES30.glTexImage2D(GLES30.GL_TEXTURE_2D, 0, GLES30.GL_RGBA,
                    mImageWidth, mImageHeight, 0, GLES30.GL_RGBA, GLES30.GL_UNSIGNED_BYTE, null);
            GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                    GLES30.GL_TEXTURE_MAG_FILTER, GLES30.GL_LINEAR);
            GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                    GLES30.GL_TEXTURE_MIN_FILTER, GLES30.GL_LINEAR);
            GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                    GLES30.GL_TEXTURE_WRAP_S, GLES30.GL_CLAMP_TO_EDGE);
            GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                    GLES30.GL_TEXTURE_WRAP_T, GLES30.GL_CLAMP_TO_EDGE);

            GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, mFramebuffers[i]);
            GLES30.glFramebufferTexture2D(GLES30.GL_FRAMEBUFFER, GLES30.GL_COLOR_ATTACHMENT0,
                    GLES30.GL_TEXTURE_2D, mFrameBufferTextures[i], 0);

            GLES30.glBindTexture(GLES30.GL_TEXTURE_2D, 0);
            GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, 0);
        }
    }

    /**
     * 销毁Framebuffers
     */
    public void destroyFramebuffer() {
        if (mFrameBufferTextures != null) {
            GLES30.glDeleteTextures(mFrameBufferTextures.length, mFrameBufferTextures, 0);
            mFrameBufferTextures = null;
        }
        if (mFramebuffers != null) {
            GLES30.glDeleteFramebuffers(mFramebuffers.length, mFramebuffers, 0);
            mFramebuffers = null;
        }
    }
```
BaseImageFilterGroup 主要用于管理FBO的创建和销毁。其中，onInputSizeChanged 方法主要用于相机预览大小(PreviewSize)改变时调用，onDisplayChanged 方法则用于SurfaceView大小发生改变时调用，initFramebuffer 方法用于为每个滤镜创建一个FBO，destroyFramebuffer 方法则用于销毁FBO。drawFrame 方法继承于基类BaseImageFilter，需要重写该方法，逐个滤镜绑定FBO并绘制。

这样，所有滤镜和滤镜组写起来就方便很多了。我们来看下其中一个滤镜和滤镜组的写法，
比如相机输入流绘制可以这么写：
```
public class CameraFilter extends BaseImageFilter {
    private static final String VERTEX_SHADER =
            "uniform mat4 uMVPMatrix;                               \n" +
            "uniform mat4 uTexMatrix;                               \n" +
            "attribute vec4 aPosition;                              \n" +
            "attribute vec4 aTextureCoord;                          \n" +
            "varying vec2 textureCoordinate;                            \n" +
            "void main() {                                          \n" +
            "    gl_Position = uMVPMatrix * aPosition;              \n" +
            "    textureCoordinate = (uTexMatrix * aTextureCoord).xy;   \n" +
            "}                                                      \n";

    private static final String FRAGMENT_SHADER_OES =
            "#extension GL_OES_EGL_image_external : require         \n" +
            "precision mediump float;                               \n" +
            "varying vec2 textureCoordinate;                            \n" +
            "uniform samplerExternalOES inputTexture;                   \n" +
            "void main() {                                          \n" +
            "    gl_FragColor = texture2D(inputTexture, textureCoordinate); \n" +
            "}                                                      \n";


    private int muTexMatrixLoc;
    private float[] mTextureMatrix;

    public CameraFilter() {
        this(VERTEX_SHADER, FRAGMENT_SHADER_OES);
    }

    public CameraFilter(String vertexShader, String fragmentShader) {
        super(vertexShader, fragmentShader);
        muTexMatrixLoc = GLES30.glGetUniformLocation(mProgramHandle, "uTexMatrix");
        // 视图矩阵
        Matrix.setLookAtM(mViewMatrix, 0, 0, 0, -1, 0f, 0f, 0f, 0f, 1f, 0f);
    }

    @Override
    public void onInputSizeChanged(int width, int height) {
        super.onInputSizeChanged(width, height);
        float aspect = (float) width / height; // 计算宽高比
        Matrix.perspectiveM(mProjectionMatrix, 0, 60, aspect, 2, 10);
    }

    @Override
    public int getTextureType() {
        return GLES11Ext.GL_TEXTURE_EXTERNAL_OES;
    }

    @Override
    public void onDrawArraysBegin() {
        GLES30.glUniformMatrix4fv(muTexMatrixLoc, 1, false, mTextureMatrix, 0);
    }

    public void updateTextureBuffer() {
        mTexCoordArray = TextureRotationUtils.getTextureBuffer();
    }

    /**
     * 设置SurfaceTexture的变换矩阵
     * @param texMatrix
     */
    public void setTextureTransformMatirx(float[] texMatrix) {
        mTextureMatrix = texMatrix;
    }

    /**
     * 镜像翻转
     * @param coords
     * @param matrix
     * @return
     */
    private float[] transformTextureCoordinates(float[] coords, float[] matrix) {
        float[] result = new float[coords.length];
        float[] vt = new float[4];

        for (int i = 0; i < coords.length; i += 2) {
            float[] v = { coords[i], coords[i + 1], 0, 1 };
            Matrix.multiplyMV(vt, 0, matrix, 0, v, 0);
            result[i] = vt[0];// x轴镜像
            // result[i + 1] = vt[1];y轴镜像
            result[i + 1] = coords[i + 1];
        }
        return result;
    }
}
```
而滤镜组则更加简单：
```
public class DefaultFilterGroup extends BaseImageFilterGroup {
    // 实时美颜层
    private static final int BeautyfyIndex = 0;
    // 颜色层
    private static final int ColorIndex = 1;
    // 瘦脸大眼层
    private static final int FaceStretchIndex = 2;
    // 贴纸
    private static final int StickersIndex = 3;

    public DefaultFilterGroup() {
        this(initFilters());
    }

    private DefaultFilterGroup(List<BaseImageFilter> filters) {
        mFilters = filters;
    }

    private static List<BaseImageFilter> initFilters() {
        List<BaseImageFilter> filters = new ArrayList<BaseImageFilter>();
        filters.add(BeautyfyIndex, FilterManager.getFilter(FilterType.REALTIMEBEAUTY));
        filters.add(ColorIndex, FilterManager.getFilter(FilterType.SOURCE));
        filters.add(FaceStretchIndex, FilterManager.getFilter(FilterType.FACESTRETCH));
        filters.add(StickersIndex, FilterManager.getFilter(FilterType.STICKER));
        return filters;
    }

    @Override
    public void changeFilter(FilterType type) {
        FilterIndex index = FilterManager.getIndex(type);
        if (index == FilterIndex.BeautyIndex) {
            changeBeautyFilter(type);
        } else if (index == FilterIndex.ColorIndex) {
            changeColorFilter(type);
        } else if (index == FilterIndex.FaceStretchIndex) {
            changeFaceStretchFilter(type);
        } else if (index == FilterIndex.MakeUpIndex) {
            changeMakeupFilter(type);
        } else if (index == FilterIndex.StickerIndex) {
            changeStickerFilter(type);
        }
    }

    /**
     * 切换美颜滤镜
     * @param type
     */
    private void changeBeautyFilter(FilterType type) {
        if (mFilters != null) {
            mFilters.get(BeautyfyIndex).release();
            mFilters.set(BeautyfyIndex, FilterManager.getFilter(type));
            // 设置宽高
            mFilters.get(BeautyfyIndex).onInputSizeChanged(mImageWidth, mImageHeight);
            mFilters.get(BeautyfyIndex).onDisplayChanged(mDisplayWidth, mDisplayHeight);
        }
    }

    /**
     * 切换颜色滤镜
     * @param type
     */
    private void changeColorFilter(FilterType type) {
        if (mFilters != null) {
            mFilters.get(ColorIndex).release();
            mFilters.set(ColorIndex, FilterManager.getFilter(type));
            // 设置宽高
            mFilters.get(ColorIndex).onInputSizeChanged(mImageWidth, mImageHeight);
            mFilters.get(ColorIndex).onDisplayChanged(mDisplayWidth, mDisplayHeight);
        }
    }

    /**
     * 切换瘦脸大眼滤镜
     * @param type
     */
    private void changeFaceStretchFilter(FilterType type) {
        if (mFilters != null) {
            mFilters.get(FaceStretchIndex).release();
            mFilters.set(FaceStretchIndex, FilterManager.getFilter(type));
            // 设置宽高
            mFilters.get(FaceStretchIndex).onInputSizeChanged(mImageWidth, mImageHeight);
            mFilters.get(FaceStretchIndex).onDisplayChanged(mDisplayWidth, mDisplayHeight);
        }
    }

    /**
     * 切换贴纸滤镜
     * @param type
     */
    private void changeStickerFilter(FilterType type) {
        if (mFilters != null) {
            mFilters.get(StickersIndex).release();
            mFilters.set(StickersIndex, FilterManager.getFilter(type));
            // 设置宽高
            mFilters.get(StickersIndex).onInputSizeChanged(mImageWidth, mImageHeight);
            mFilters.get(StickersIndex).onDisplayChanged(mDisplayWidth, mDisplayHeight);
        }
    }

    /**
     * 切换彩妆滤镜
     * @param type
     */
    private void changeMakeupFilter(FilterType type) {
        // Do nothing, 彩妆滤镜放在彩妆滤镜组里面
    }
}
```
上面是默认的实时渲染滤镜组，可以看到，本人在处理多个滤镜的时候，使用了分层的形式。这么写的原因很简单，为了节省渲染前的计算时间。一般实时性不够强的情况下，通常都是在调用某个方法的时候再逐个计算有多少个滤镜，然后在绘制的时候再动态绑定FBO。但这么做对实时性要求非常高的情况下并不是非常好的方式。因为每次调用drawFrame/drawFrameBuffer绘制的时候都需要绑定和解绑FBO，效率肯定会有所影响。但实际上，我们可以在绘制之前就已经知道需要多少个滤镜，需要多少个FBO了，绘制方法因为调用的次数非常非常多，所以任何耗时操作如非必须，都不能放在绘制方法里面，新建对象就更加不建议了，不仅新建对象比较耗时，而且可能会产生内存抖动问题。

现在我们得到了滤镜和滤镜组的基类，也得到了默认实时渲染的滤镜组，接下来就是实现具体滤镜了。具体的滤镜实现就是glsl的事情了。如果滤镜有多个Texture如何处理呢？没关系，我们在BaseImageFilter的drawFrame 方法中加入了一个空方法onDrawArraysBegin()，当滤镜需要绑定除了inputTexture外的其他Texture，则可以在这里进行绑定，比如像LOMO滤镜一样，绑定需要混合的Texture：
```
public class LomoFilter extends BaseImageFilter {

    private static final String FRAGMENT_SHADER =
            "precision mediump float;\n" +
            " \n" +
            " varying mediump vec2 textureCoordinate;\n" +
            " \n" +
            " uniform sampler2D inputTexture;\n" +
            " uniform sampler2D mapTexture;\n" +
            " uniform sampler2D vignetteTexture;\n" +
            " \n" +
            " uniform float strength;\n" +
            "\n" +
            " void main()\n" +
            " {\n" +
            "     vec4 originColor = texture2D(inputTexture, textureCoordinate);\n" +
            "     vec3 texel = texture2D(inputTexture, textureCoordinate).rgb;\n" +
            "\n" +
            "     vec2 red = vec2(texel.r, 0.16666);\n" +
            "     vec2 green = vec2(texel.g, 0.5);\n" +
            "     vec2 blue = vec2(texel.b, 0.83333);\n" +
            "\n" +
            "     texel.rgb = vec3(\n" +
            "                      texture2D(mapTexture, red).r,\n" +
            "                      texture2D(mapTexture, green).g,\n" +
            "                      texture2D(mapTexture, blue).b);\n" +
            "\n" +
            "     vec2 tc = (2.0 * textureCoordinate) - 1.0;\n" +
            "     float d = dot(tc, tc);\n" +
            "     vec2 lookup = vec2(d, texel.r);\n" +
            "     texel.r = texture2D(vignetteTexture, lookup).r;\n" +
            "     lookup.y = texel.g;\n" +
            "     texel.g = texture2D(vignetteTexture, lookup).g;\n" +
            "     lookup.y = texel.b;\n" +
            "     texel.b\t= texture2D(vignetteTexture, lookup).b;\n" +
            "\n" +
            "     texel.rgb = mix(originColor.rgb, texel.rgb, strength);\n" +
            "\n" +
            "     gl_FragColor = vec4(texel,1.0);\n" +
            " }";


    private int mMapTexture;
    private int mMapTextureLoc;

    private int mVignetteTexture;
    private int mVignetteTextureLoc;

    private int mStrengthLoc;

    public LomoFilter() {
        this(VERTEX_SHADER, FRAGMENT_SHADER);
    }

    public LomoFilter(String vertexShader, String fragmentShader) {
        super(vertexShader, fragmentShader);

        mMapTextureLoc = GLES30.glGetUniformLocation(mProgramHandle, "mapTexture");
        mVignetteTextureLoc = GLES30.glGetUniformLocation(mProgramHandle, "vignetteTexture");
        mStrengthLoc = GLES30.glGetUniformLocation(mProgramHandle, "strength");
        createTexture();
        setFloat(mStrengthLoc, 1.0f);
    }

    private void createTexture() {
        mMapTexture = GlUtil.createTextureFromAssets(ParamsManager.context,
                "filters/lomo_map.png");
        mVignetteTexture = GlUtil.createTextureFromAssets(ParamsManager.context,
                "filters/lomo_vignette.png");
    }

    @Override
    public void onDrawArraysBegin() {
        super.onDrawArraysBegin();
        GLES30.glActiveTexture(GLES30.GL_TEXTURE1);
        GLES30.glBindTexture(getTextureType(), mMapTexture);
        GLES30.glUniform1i(mMapTextureLoc, 1);

        GLES30.glActiveTexture(GLES30.GL_TEXTURE2);
        GLES30.glBindTexture(getTextureType(), mVignetteTexture);
        GLES30.glUniform1i(mVignetteTextureLoc, 2);
    }

    @Override
    public void release() {
        super.release();
        GLES30.glDeleteTextures(2, new int[]{mMapTexture, mVignetteTexture}, 0);
    }
}
```
滤镜之所以没有持有Context，这样做可以让RenderThread也不需要持有相应的Context，方便移植到不同的系统中，如果有需要，也可以在基类添加Context上下文。这就看个人喜好了。 
好了，滤镜的实现我们也知道怎么做了。那么，像我这样将滤镜组分层又怎么做滤镜切换呢？为了方便，我写了几个管理类，用于管理滤镜和不同滤镜组的切换，其中FilterIndex用于指定滤镜层，FilterType用于指定具体的滤镜，FilterManager用于管理切换滤镜以及切换滤镜组，ColorFilterManager则用于管理颜色滤镜的切换，方便Activity调用。具体实现如下：
FilterIndex：
```
public enum FilterIndex {
    // 无
    NoneIndex,

    // 美颜
    BeautyIndex,

    // 颜色
    ColorIndex,

    // 瘦脸大眼
    FaceStretchIndex,

    // 贴纸
    StickerIndex,

    // 彩妆
    MakeUpIndex,

    // 水印
    WaterMaskIndex,

    // 图片编辑
    ImageEditIndex
}
``
FilterType:
```
public enum FilterType {
    NONE, // 没有滤镜

    // 图片编辑滤镜
    BRIGHTNESS, // 亮度
    CONTRAST, // 对比度
    EXPOSURE, // 曝光
    GUASS, // 高斯模糊
    HUE, // 色调
    MIRROR, // 镜像
    SATURATION, // 饱和度
    SHARPNESS, // 锐度

    WATERMASK, // 水印

    // 人脸美颜美妆贴纸
    REALTIMEBEAUTY, // 实时美颜
    FACESTRETCH, // 人脸变形(瘦脸大眼等)

    STICKER,    // 贴纸
    MAKEUP, // 彩妆

    // 颜色滤镜
    SOURCE,         // 原图
    AMARO,          // 阿马罗
    ANTIQUE,        // 古董
    BLACKCAT,       // 黑猫
    BLACKWHITE,     // 黑白
    BROOKLYN,       // 布鲁克林
    CALM,           // 冷静
    COOL,           // 冷色调
    EARLYBIRD,      // 晨鸟
    EMERALD,        // 翡翠
    EVERGREEN,      // 常绿
    FAIRYTALE,      // 童话
    FREUD,          // 佛洛伊特
    HEALTHY,        // 健康
    HEFE,           // 酵母
    HUDSON,         // 哈德森
    KEVIN,          // 凯文
    LATTE,          // 拿铁
    LOMO,           // LOMO
    NOSTALGIA,      // 怀旧之情
    ROMANCE,        // 浪漫
    SAKURA,         // 樱花
    SKETCH,         // 素描
    SUNSET,         // 日落
    WHITECAT,       // 白猫
    WHITENORREDDEN, // 白皙还是红润
}
```
FilterManager:
```
public final class FilterManager {

    private static HashMap<FilterType, FilterIndex> mIndexMap = new HashMap<FilterType, FilterIndex>();
    static {
        mIndexMap.put(FilterType.NONE, FilterIndex.NoneIndex);

        // 图片编辑
        mIndexMap.put(FilterType.BRIGHTNESS, FilterIndex.ImageEditIndex);
        mIndexMap.put(FilterType.CONTRAST, FilterIndex.ImageEditIndex);
        mIndexMap.put(FilterType.EXPOSURE, FilterIndex.ImageEditIndex);
        mIndexMap.put(FilterType.GUASS, FilterIndex.ImageEditIndex);
        mIndexMap.put(FilterType.HUE, FilterIndex.ImageEditIndex);
        mIndexMap.put(FilterType.MIRROR, FilterIndex.ImageEditIndex);
        mIndexMap.put(FilterType.SATURATION, FilterIndex.ImageEditIndex);
        mIndexMap.put(FilterType.SHARPNESS, FilterIndex.ImageEditIndex);

        // 水印
        mIndexMap.put(FilterType.WATERMASK, FilterIndex.WaterMaskIndex);

        // 美颜
        mIndexMap.put(FilterType.REALTIMEBEAUTY, FilterIndex.BeautyIndex);

        // 瘦脸大眼
        mIndexMap.put(FilterType.FACESTRETCH, FilterIndex.FaceStretchIndex);

        // 贴纸
        mIndexMap.put(FilterType.STICKER, FilterIndex.StickerIndex);

        // 彩妆
        mIndexMap.put(FilterType.MAKEUP, FilterIndex.MakeUpIndex);


        // 颜色滤镜
        mIndexMap.put(FilterType.AMARO, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.ANTIQUE, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.BLACKCAT, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.BLACKWHITE, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.BROOKLYN, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.CALM, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.COOL, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.EARLYBIRD, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.EMERALD, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.EVERGREEN, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.FAIRYTALE, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.FREUD, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.HEALTHY, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.HEFE, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.HUDSON, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.KEVIN, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.LATTE, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.LOMO, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.NOSTALGIA, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.ROMANCE, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.SAKURA, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.SKETCH, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.SOURCE, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.SUNSET, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.WHITECAT, FilterIndex.ColorIndex);
        mIndexMap.put(FilterType.WHITENORREDDEN, FilterIndex.ColorIndex);
    }

    private FilterManager() {}

    public static BaseImageFilter getFilter(FilterType type) {
        switch (type) {

            // 图片基本属性编辑滤镜
            // 饱和度
            case SATURATION:
                return new SaturationFilter();
            // 镜像翻转
            case MIRROR:
                return new MirrorFilter();
            // 高斯模糊
            case GUASS:
                return new GuassFilter();
            // 亮度
            case BRIGHTNESS:
                return new BrightnessFilter();
            // 对比度
            case CONTRAST:
                return new ContrastFilter();
            // 曝光
            case EXPOSURE:
                return new ExposureFilter();
            // 色调
            case HUE:
                return new HueFilter();
            // 锐度
            case SHARPNESS:
                return new SharpnessFilter();

            // TODO 贴纸滤镜需要人脸关键点计算得到
            case STICKER:
                return new DisplayFilter();
//                return new StickerFilter();

            // 白皙还是红润
            case WHITENORREDDEN:
                return new WhitenOrReddenFilter();
            // 实时磨皮
            case REALTIMEBEAUTY:
                return new RealtimeBeautify();

            // AMARO
            case AMARO:
                return new AmaroFilter();
            // 古董
            case ANTIQUE:
                return new AnitqueFilter();

            // 黑猫
            case BLACKCAT:
                return new BlackCatFilter();

            // 黑白
            case BLACKWHITE:
                return new BlackWhiteFilter();

            // 布鲁克林
            case BROOKLYN:
                return new BrooklynFilter();

            // 冷静
            case CALM:
                return new CalmFilter();

            // 冷色调
            case COOL:
                return new CoolFilter();

            // 晨鸟
            case EARLYBIRD:
                return new EarlyBirdFilter();

            // 翡翠
            case EMERALD:
                return new EmeraldFilter();

            // 常绿
            case EVERGREEN:
                return new EvergreenFilter();

            // 童话
            case FAIRYTALE:
                return new FairyTaleFilter();

            // 佛洛伊特
            case FREUD:
                return new FreudFilter();

            // 健康
            case HEALTHY:
                return new HealthyFilter();

            // 酵母
            case HEFE:
                return new HefeFilter();

            // 哈德森
            case HUDSON:
                return new HudsonFilter();

            // 凯文
            case KEVIN:
                return new KevinFilter();

            // 拿铁
            case LATTE:
                return new LatteFilter();

            // LOMO
            case LOMO:
                return new LomoFilter();

            // 怀旧之情
            case NOSTALGIA:
                return new NostalgiaFilter();

            // 浪漫
            case ROMANCE:
                return new RomanceFilter();

            // 樱花
            case SAKURA:
                return new SakuraFilter();

            //  素描
            case SKETCH:
                return new SketchFilter();

            // 日落
            case SUNSET:
                return new SunsetFilter();

            // 白猫
            case WHITECAT:
                return new WhiteCatFilter();

            case NONE:      // 没有滤镜
            case SOURCE:    // 原图
            default:
                return new DisplayFilter();
        }
    }

    /**
     * 获取滤镜组
     * @return
     */
    public static BaseImageFilterGroup getFilterGroup() {
        return new DefaultFilterGroup();
    }

    public static BaseImageFilterGroup getFilterGroup(FilterGroupType type) {
        switch (type) {
            // 彩妆滤镜组
            case MAKEUP:
                return new MakeUpFilterGroup();

            // 默认滤镜组
            case DEFAULT:
            default:
                return new DefaultFilterGroup();
        }
    }

    /**
     * 获取层级
     * @param Type
     * @return
     */
    public static FilterIndex getIndex(FilterType Type) {
        FilterIndex index = mIndexMap.get(Type);
        if (index != null) {
            return index;
        }
        return FilterIndex.NoneIndex;
    }
}
```
ColorFilterManager:
```
public final class ColorFilterManager {

    private static ColorFilterManager mInstance;


    private ArrayList<FilterType> mFilterType;
    private ArrayList<String> mFilterName;

    public static ColorFilterManager getInstance() {
        if (mInstance == null) {
            mInstance = new ColorFilterManager();
        }
        return mInstance;
    }

    private ColorFilterManager() {
        initColorFilters();
    }


    /**
     * 初始化颜色滤镜
     */
    public void initColorFilters() {
        mFilterType = new ArrayList<FilterType>();

        mFilterType.add(FilterType.SOURCE); // 原图
        mFilterType.add(FilterType.AMARO);
        mFilterType.add(FilterType.ANTIQUE);
        mFilterType.add(FilterType.BLACKCAT);
        mFilterType.add(FilterType.BLACKWHITE);
        mFilterType.add(FilterType.BROOKLYN);
        mFilterType.add(FilterType.CALM);
        mFilterType.add(FilterType.COOL);
        mFilterType.add(FilterType.EARLYBIRD);
        mFilterType.add(FilterType.EMERALD);
        mFilterType.add(FilterType.EVERGREEN);
        mFilterType.add(FilterType.FAIRYTALE);
        mFilterType.add(FilterType.FREUD);
        mFilterType.add(FilterType.HEALTHY);
        mFilterType.add(FilterType.HEFE);
        mFilterType.add(FilterType.HUDSON);
        mFilterType.add(FilterType.KEVIN);
        mFilterType.add(FilterType.LATTE);
        mFilterType.add(FilterType.LOMO);
        mFilterType.add(FilterType.NOSTALGIA);
        mFilterType.add(FilterType.ROMANCE);
        mFilterType.add(FilterType.SAKURA);
        mFilterType.add(FilterType.SKETCH);
        mFilterType.add(FilterType.SUNSET);
        mFilterType.add(FilterType.WHITECAT);



        mFilterName = new ArrayList<String>();
        mFilterName.add("原图");
        mFilterName.add("阿马罗");
        mFilterName.add("古董");
        mFilterName.add("黑猫");
        mFilterName.add("黑白");
        mFilterName.add("布鲁克林");
        mFilterName.add("冷静");
        mFilterName.add("冷色调");
        mFilterName.add("晨鸟");
        mFilterName.add("翡翠");
        mFilterName.add("常绿");
        mFilterName.add("童话");
        mFilterName.add("佛洛伊特");
        mFilterName.add("健康");
        mFilterName.add("酵母");
        mFilterName.add("哈德森");
        mFilterName.add("凯文");
        mFilterName.add("拿铁");
        mFilterName.add("LOMO");
        mFilterName.add("怀旧之情");
        mFilterName.add("浪漫");
        mFilterName.add("樱花");
        mFilterName.add("素描");
        mFilterName.add("日落");
        mFilterName.add("白猫");

    }

    /**
     * 获取颜色滤镜类型
     * @param index
     * @return
     */
    public FilterType getColorFilterType(int index) {
        if (mFilterType == null || mFilterType.isEmpty()) {
            return FilterType.SOURCE;
        }
        int i = index % mFilterType.size();
        return mFilterType.get(i);
    }

    /**
     * 获取颜色滤镜的名称
     * @param index
     * @return
     */
    public String getColorFilterName(int index) {
        if (mFilterName == null || mFilterName.isEmpty()) {
            return "原图";
        }
        int i = index % mFilterName.size();
        return mFilterName.get(i);
    }

    /**
     * 获取颜色滤镜数目
     * @return
     */
    public int getColorFilterCount() {
        return mFilterType == null ? 0 : mFilterType.size();
    }
}
```
这样，在外层的Activity中，切换滤镜只需要这么使用：
```
    @Override
    public void swipeBack() {
        mColorIndex++;
        if (mColorIndex >= ColorFilterManager.getInstance().getColorFilterCount()) {
            mColorIndex = 0;
        }
        DrawerManager.getInstance()
                .changeFilterType(ColorFilterManager.getInstance().getColorFilterType(mColorIndex));
        if (isDebug) {
            Log.d("changeFilter", "index = " + mColorIndex + ", filter name = "
                    + ColorFilterManager.getInstance().getColorFilterName(mColorIndex));
        }
    }

    @Override
    public void swipeFrontal() {
        mColorIndex--;
        if (mColorIndex < 0) {
            int count = ColorFilterManager.getInstance().getColorFilterCount();
            mColorIndex = count > 0 ? count - 1 : 0;
        }
        DrawerManager.getInstance()
                .changeFilterType(ColorFilterManager.getInstance().getColorFilterType(mColorIndex));
        if (isDebug) {
            Log.d("changeFilter", "index = " + mColorIndex + ", filter name = "
                    + ColorFilterManager.getInstance().getColorFilterName(mColorIndex));
        }
    }
``
其中DrawerManager是用于管理相机预览渲染线程的。这样的方案可以使得我们在操作页面逻辑的时候就不需要考虑滤镜切换的细节是如何处理的，页面需要大改的时候，对滤镜来说并没有任何影响。

至此，你就会得到一个耦合度较低、渲染效率较高、预览帧率足够的美颜类相机了。

具体实现过程可以参考本人的项目CainCamera，github地址：[CainCamera](https://github.com/CainKernel/CainCamera)

备注：截止这篇文章发布时，本人还没有实现贴纸功能，但人脸关键点检测已经成功接入Face++ 的SDK，关键点已经取得，但贴纸滤镜的具体实现还没有实现。本人的实现思路如下：
 贴纸功能分成JSON解析器、zip解压器、贴纸管理器等功能构成，用于解压贴纸的zip包，解析json，生成texture等流程，后续功能等实现了，本人将会发布新的文章介绍如何实现。当然，这是在帧率得到保证、不产生内存抖动、内存溢出的前提下实现。本人对CainCamera项目的要求能够达到商业相机的水平，直接可以商用的程度。

关于相机控制和渲染具体实现，可以参考本人的文章：
[Android预览实时渲染的帧率优化相关](http://www.jianshu.com/p/7187fe10261d)
关于磨皮算法的渲染效率优化，可以参考本人的文章：
[Android OpenGLES 实时美颜的优化](http://www.jianshu.com/p/a76a1201ae53)



