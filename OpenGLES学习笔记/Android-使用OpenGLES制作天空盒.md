我们都知道，像Unity3D等游戏引擎都有许多天空盒的资源，通过天空盒，我们能够制作许多精美的场景，比如我们玩游戏的背景，跟随玩家的视点的转动而转动。这样的场景我们再熟悉不过了。那么，下面开始介绍使用OpenGLES来制作一个天空盒。
首先，在制作天空盒之前我们需要准备一些天空盒的资源，这里有个网址，上面有许多天空盒的资源供我们做实验：
http://www.custommapmakers.org/skyboxes.php

此外，制作天空盒，我们需要知道天空盒的原理。天空盒的原理就是在三维空间中放置一个正方体，然后将我们的相机放置在正方体内，当我们的视点转动，相机跟着转动。我们就可以看到相应的景色的变换了。那么OpenGLES 中，怎么实现天空盒呢？天空盒本质上是一个立方体，通过查找资源，我们可以知道OpenGLES 是支持GL_TEXTURE_CUBE_MAP的Texture的。GL_TEXTURE_CUBE_MAP是有六个mipmap组成的Texture，分别是：
左 —— GL_TEXTURE_CUBE_MAP_NEGATIVE_X
右 —— GL_TEXTURE_CUBE_MAP_POSITIVE_X
下 —— GL_TEXTURE_CUBE_MAP_NEGATIVE_Y
上 —— GL_TEXTURE_CUBE_MAP_POSITIVE_Y
前 —— GL_TEXTURE_CUBE_MAP_NEGATIVE_Z
后 —— GL_TEXTURE_CUBE_MAP_NEGATIVE_Z

我们可以通过给这六个面设置不同的mipmap贴图，就可以实现天空盒的效果了，
废话不多说，直接进入主题吧：
1、新建一个Activity，并将GLSurfaceView加入到ContentView中，然后加入方向向量传感器的监听，代码如下：
```
public class SkyBoxActivity extends Activity implements  SensorEventListener {

    private GLSurfaceView mGlSurfaceView;
    private SkyBoxRenderer mSkyBoxRenderer;
    private boolean rendererSet = false;

    private SensorManager mSensorManager;
    private Sensor mRotationSensor;
    private float[] mRotationMatrix = new float[16];

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        mGlSurfaceView = new GLSurfaceView(this);

        ActivityManager activityManager = 
            (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
        ConfigurationInfo configurationInfo = activityManager
            .getDeviceConfigurationInfo();
        final boolean supportsEs2 =
            configurationInfo.reqGlEsVersion >= 0x20000
                || (Build.VERSION.SDK_INT >= Build.VERSION_CODES.ICE_CREAM_SANDWICH_MR1
                 && (Build.FINGERPRINT.startsWith("generic")
                  || Build.FINGERPRINT.startsWith("unknown")
                  || Build.MODEL.contains("google_sdk")
                  || Build.MODEL.contains("Emulator")
                  || Build.MODEL.contains("Android SDK built for x86")));

        mSkyBoxRenderer = new SkyBoxRenderer(this);

        if (supportsEs2) {
            mGlSurfaceView.setEGLContextClientVersion(2);
            mGlSurfaceView.setRenderer(mSkyBoxRenderer);
            rendererSet = true;
        } else {
            Toast.makeText(this, "This device does not support OpenGL ES 2.0.",
                Toast.LENGTH_LONG).show();
            return;
        }


        setContentView(mGlSurfaceView);

        // 方向向量s
        mSensorManager = (SensorManager)getSystemService(Context.SENSOR_SERVICE);
        mRotationSensor = mSensorManager.getDefaultSensor(Sensor.TYPE_ROTATION_VECTOR);
        Matrix.setIdentityM(mRotationMatrix, 0);
    }
    
    @Override
    protected void onPause() {
        super.onPause();
        
        if (rendererSet) {
            mGlSurfaceView.onPause();
        }

        mSensorManager.unregisterListener(this);
    }

    @Override
    protected void onResume() {
        super.onResume();
        
        if (rendererSet) {
            mGlSurfaceView.onResume();
        }

        mSensorManager.registerListener(this, mRotationSensor, SensorManager.SENSOR_DELAY_GAME);
    }

    @Override
    public void onSensorChanged(SensorEvent event) {
        SensorManager.getRotationMatrixFromVector(mRotationMatrix, event.values);
        if (mGlSurfaceView != null) {
            mGlSurfaceView.queueEvent(new Runnable() {
                @Override
                public void run() {
                    mSkyBoxRenderer.setRotationMatrix(mRotationMatrix);
                }
            });
        }
    }

    @Override
    public void onAccuracyChanged(Sensor sensor, int accuracy) {

    }
}
```
接下来我们来看下SkyRenderer类的实现：
```
public class SkyBoxRenderer implements GLSurfaceView.Renderer {
    private final Activity mContext;

    private int mViewWidth;
    private int mViewHeight;

    // 天空盒滤镜
    private SkyBoxFilter mSkyBoxFilter;


    public SkyBoxRenderer(Activity context) {
        mContext = context;

    }

    /**
     * 设置旋转矩阵
     * @param matrix
     */
    public void setRotationMatrix(float[] matrix) {
        if (mSkyBoxFilter != null) {
            mSkyBoxFilter.setRotationMatrix(matrix);
        }
    }

    @Override
    public void onSurfaceCreated(GL10 glUnused, EGLConfig config) {
        GLES20.glClearColor(0.0f, 0.0f, 0.0f, 0.0f);

        // 创建天空盒
        mSkyBoxFilter = new SkyBoxFilter(mContext);
        mSkyBoxFilter.createProgram();

        
    }

    @Override
    public void onSurfaceChanged(GL10 glUnused, int width, int height) {                
        GLES20.glViewport(0, 0, width, height);
        mViewWidth = width;
        mViewHeight = height;
        // 调整视图大小
        if (mSkyBoxFilter != null) {
            mSkyBoxFilter.setViewSize(width, height);
        }
        
    }

    @Override    
    public void onDrawFrame(GL10 glUnused) {
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
        // 绘制天空盒
        drawSkybox();
        
    }


    /**
     * 绘制天空盒
     */
    private void drawSkybox() {
        if (mSkyBoxFilter != null) {
            mSkyBoxFilter.drawSkyBox();
        }
    }
}
```
Renderer类主要是调用了SkyBoxFilter类实现天空盒的绘制，SkyBoxFilter类的实现过程如下：
```
/**
 * 天空盒滤镜
 * Created by cain.huang on 2017/8/31.
 */
public class SkyBoxFilter {

    private static final int COORDS_PER_VERTEX = 3;

    // 立方体坐标
    private static final float[] CubeCoords = new float[] {
            -1,  1,  1,     // (0) Top-left near
            1,   1,  1,     // (1) Top-right near
            -1, -1,  1,     // (2) Bottom-left near
            1,  -1,  1,     // (3) Bottom-right near
            -1,  1, -1,     // (4) Top-left far
            1,   1, -1,     // (5) Top-right far
            -1, -1, -1,     // (6) Bottom-left far
            1,  -1, -1      // (7) Bottom-right far
    };

    // 立方体索引
    private static final byte[] CubeIndex = new byte[] {
            // Front
            1, 3, 0,
            0, 3, 2,

            // Back
            4, 6, 5,
            5, 6, 7,

            // Left
            0, 2, 4,
            4, 2, 6,

            // Right
            5, 7, 1,
            1, 7, 3,

            // Top
            5, 1, 4,
            4, 1, 0,

            // Bottom
            6, 2, 7,
            7, 2, 3
    };

    private Context mContext;

    // 视图宽高
    private int mViewWidth;
    private int mViewHeight;

    private FloatBuffer mVertexBuffer;
    private ByteBuffer mIndexBuffer;

    private int mProgramHandle;

    private int muMatrixHandle;
    private int muTextureUnitHandle;
    private int maPositionHandle;

    // Cube纹理
    private int mSkyboxTexture;

    // 变换矩阵
    private float[] mRotationMatrix = new float[16]; // 旋转矩阵
    private float[] mViewMatrix = new float[16];    // 视图矩阵
    private float[] mProjectionMatrix = new float[16]; // 投影矩阵
    private float[] mMVPMatrix = new float[16]; // 总变换矩阵


    public SkyBoxFilter(Context context) {
        mContext = context;
        mVertexBuffer = GlUtil.createFloatBuffer(CubeCoords);
        mIndexBuffer = GlUtil.createByteBuffer(CubeIndex);
        initMatrix();
    }

    /**
     * 初始化Matrix
     */
    private void initMatrix() {
        Matrix.setIdentityM(mRotationMatrix, 0);
        Matrix.setIdentityM(mViewMatrix, 0);
        Matrix.setIdentityM(mProjectionMatrix, 0);
        Matrix.setIdentityM(mMVPMatrix, 0);
    }

    /**
     * 创建Program
     */
    public void createProgram() {
        mProgramHandle = GlUtil.createProgram(ResourceUtil
                        .readTextFileFromResource(mContext, R.raw.vertex_skybox),
                ResourceUtil
                        .readTextFileFromResource(mContext, R.raw.fragment_skybox));
        muMatrixHandle = glGetUniformLocation(mProgramHandle, "u_Matrix");
        muTextureUnitHandle = glGetUniformLocation(mProgramHandle, "u_TextureUnit");
        maPositionHandle = glGetAttribLocation(mProgramHandle, "a_Position");
        mSkyboxTexture =  TextureHelper.loadCubeMap(mContext,
                new int[] {
                        R.drawable.left, R.drawable.right,
                        R.drawable.bottom, R.drawable.top,
                        R.drawable.front, R.drawable.back,
                });
    }

    /**
     * 更新视图大小，用于设置视图矩阵和透视矩阵
     * @param width
     * @param height
     */
    public void setViewSize(int width, int height) {
        mViewWidth = width;
        mViewHeight = height;

        //设置相机位置
        Matrix.setLookAtM(mViewMatrix, 0,
                0.0f, 0.0f, 0.0f,
                0.0f, 0.0f, -1.0f,
                0f, 1.0f, 0.0f);

        // 设置透视矩阵
        float ratio = (float) width / (float) height;
        MatrixHelper.perspectiveM(mProjectionMatrix, 45, ratio, 1f, 300f);
    }

    /**
     * 绘制天空盒
     */
    public void drawSkyBox() {

        GLES20.glUseProgram(mProgramHandle);

        GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
        GLES20.glBindTexture(GLES20.GL_TEXTURE_CUBE_MAP, mSkyboxTexture);
        calculateMatrix();
        GLES20.glUniformMatrix4fv(muMatrixHandle, 1, false, mMVPMatrix, 0);
        GLES20.glUniform1i(muTextureUnitHandle, 0);

        GLES20.glEnableVertexAttribArray(maPositionHandle);
        GLES20.glVertexAttribPointer(maPositionHandle, COORDS_PER_VERTEX,
                GLES20.GL_FLOAT, false, 0, mVertexBuffer);
        GLES20.glDrawElements(GLES20.GL_TRIANGLES, 36, GLES20.GL_UNSIGNED_BYTE, mIndexBuffer);
        GLES20.glUseProgram(0);
    }

    /**
     * 计算总变换
     */
    private void calculateMatrix() {
        // 计算综合矩阵
        Matrix.setIdentityM(mViewMatrix, 0);
        Matrix.setLookAtM(mViewMatrix, 0,
                0.0f, 0.0f, 0.0f,
                0.0f, 0.0f, -1.0f,
                0f, 1.0f, 0.0f);
        Matrix.multiplyMM(mViewMatrix, 0, mViewMatrix, 0, mRotationMatrix, 0);
        Matrix.rotateM(mViewMatrix, 0, 90, 1f, 0f, 0f);
        Matrix.multiplyMM(mMVPMatrix, 0, mProjectionMatrix, 0, mViewMatrix, 0);
    }

    /**
     * 设置旋转矩阵
     * @param matrix
     */
    public void setRotationMatrix(float[] matrix) {
        mRotationMatrix = matrix;
    }
}
```
该类主要是创建Program、创建CubeMap的Texture、绑定GLSL的属性、绘制等。其中TextureHelper是一个用于加载CubeMap的类，实现如下：
```
public class TextureHelper {
    private static final String TAG = "TextureHelper";

    /**
     * Loads a texture from a resource ID, returning the OpenGL ID for that
     * texture. Returns 0 if the load failed.
     * 
     * @param context
     * @param resourceId
     * @return
     */
    public static int loadTexture(Context context, int resourceId) {
        final int[] textureObjectIds = new int[1];
        GLES20.glGenTextures(1, textureObjectIds, 0);
        if (textureObjectIds[0] == 0) {
            if (LoggerConfig.ON) {
                Log.w(TAG, "Could not generate a new OpenGL texture object.");
            }
            return 0;
        }
        
        final BitmapFactory.Options options = new BitmapFactory.Options();
        options.inScaled = false;

        // Read in the resource
        final Bitmap bitmap = BitmapFactory.decodeResource(
            context.getResources(), resourceId, options);

        if (bitmap == null) {
            if (LoggerConfig.ON) {
                Log.w(TAG, "Resource ID " + resourceId
                    + " could not be decoded.");
            }

            GLES20.glDeleteTextures(1, textureObjectIds, 0);

            return 0;
        } 
        
        // Bind to the texture in OpenGL
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureObjectIds[0]);

        // Set filtering: a default must be set, or the texture will be
        // black.
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR_MIPMAP_LINEAR);
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR);
        // Load the bitmap into the bound texture.
        GLUtils.texImage2D(GLES20.GL_TEXTURE_2D, 0, bitmap, 0);

        // Note: Following code may cause an error to be reported in the
        // ADB log as follows: E/IMGSRV(20095): :0: HardwareMipGen:
        // Failed to generate texture mipmap levels (error=3)
        // No OpenGL error will be encountered (glGetError() will return
        // 0). If this happens, just squash the source image to be
        // square. It will look the same because of texture coordinates,
        // and mipmap generation will work.

        GLES20.glGenerateMipmap(GLES20.GL_TEXTURE_2D);

        // Recycle the bitmap, since its data has been loaded into
        // OpenGL.
        bitmap.recycle();

        // Unbind from the texture.
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, 0);
        return textureObjectIds[0];        
    }
    /**
     * Loads a cubemap texture from the provided resources and returns the
     * texture ID. Returns 0 if the load failed.
     * 
     * @param context
     * @param cubeResources
     *            An array of resources corresponding to the cube map. Should be
     *            provided in this order: left, right, bottom, top, front, back.
     * @return
     */
    public static int loadCubeMap(Context context, int[] cubeResources) {       
        final int[] textureObjectIds = new int[1];
        GLES20.glGenTextures(1, textureObjectIds, 0);

        if (textureObjectIds[0] == 0) {
            if (LoggerConfig.ON) {
                Log.w(TAG, "Could not generate a new OpenGL texture object.");
            }
            return 0;
        }      
        final BitmapFactory.Options options = new BitmapFactory.Options();
        options.inScaled = false;
        final Bitmap[] cubeBitmaps = new Bitmap[6];
        for (int i = 0; i < 6; i++) {
            cubeBitmaps[i] =
                BitmapFactory.decodeResource(context.getResources(),
                    cubeResources[i], options);

            if (cubeBitmaps[i] == null) {
                if (LoggerConfig.ON) {
                    Log.w(TAG, "Resource ID " + cubeResources[i]
                        + " could not be decoded.");
                }
                GLES20.glDeleteTextures(1, textureObjectIds, 0);
                return 0;
            }
        }
        // Linear filtering for minification and magnification
        GLES20.glBindTexture(GLES20.GL_TEXTURE_CUBE_MAP, textureObjectIds[0]);

        GLES20.glTexParameteri(GLES20.GL_TEXTURE_CUBE_MAP, GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR);
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_CUBE_MAP, GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR);
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_CUBE_MAP, GLES20.GL_TEXTURE_WRAP_S, GLES20.GL_CLAMP_TO_EDGE);
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_CUBE_MAP, GLES20.GL_TEXTURE_WRAP_T, GLES20.GL_CLAMP_TO_EDGE);


        GLUtils.texImage2D(GLES20.GL_TEXTURE_CUBE_MAP_NEGATIVE_X, 0, cubeBitmaps[0], 0); // 左
        GLUtils.texImage2D(GLES20.GL_TEXTURE_CUBE_MAP_POSITIVE_X, 0, cubeBitmaps[1], 0); // 右

        GLUtils.texImage2D(GLES20.GL_TEXTURE_CUBE_MAP_NEGATIVE_Y, 0, cubeBitmaps[2], 0); // 下
        GLUtils.texImage2D(GLES20.GL_TEXTURE_CUBE_MAP_POSITIVE_Y, 0, cubeBitmaps[3], 0); // 上

        GLUtils.texImage2D(GLES20.GL_TEXTURE_CUBE_MAP_NEGATIVE_Z, 0, cubeBitmaps[4], 0); // 前
        GLUtils.texImage2D(GLES20.GL_TEXTURE_CUBE_MAP_POSITIVE_Z, 0, cubeBitmaps[5], 0); // 后
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, 0);

        // 释放图片
        for (Bitmap bitmap : cubeBitmaps) {
            bitmap.recycle();
        }

        return textureObjectIds[0];        
    }
}
```
接下来我们来看看GLSL的代码：
VertexShader：
```
uniform mat4 u_Matrix;
attribute vec3 a_Position;
varying vec3 v_Position;

void main()                    
{                                	  	          
    v_Position = a_Position;	
    // Make sure to convert from the right-handed coordinate system of the
    // world to the left-handed coordinate system of the cube map, otherwise,
    // our cube map will still work but everything will be flipped.
    v_Position.z = -v_Position.z;

    gl_Position = u_Matrix * vec4(a_Position, 1.0);
    gl_Position = gl_Position.xyww;
}  
```
FragmentShader:
```
precision mediump float; 

uniform samplerCube u_TextureUnit;
varying vec3 v_Position;
	    	   								
void main()                    		
{
	gl_FragColor = textureCube(u_TextureUnit, v_Position);    
}
```
大致的过程就是这样。整个过程都非常简单，主要是要理解相机的视点(视图矩阵)、CubeMap的应用等内容。整个过程与一般的OpenGLES 开发并没有任何差异。
如果没有意外，我们就能看到天空盒的效果了，转动手机，就能够看到不同的位置的贴图了：

![Screenshot_20170831-171555.png](http://upload-images.jianshu.io/upload_images/2103804-55bbe058b851f3eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Screenshot_20170831-171559.png](http://upload-images.jianshu.io/upload_images/2103804-7050fa39f03b0fdc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Screenshot_20170831-171614.png](http://upload-images.jianshu.io/upload_images/2103804-368e6afbb00ddf8a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

项目地址：
https://github.com/CainKernel/SkyBox
