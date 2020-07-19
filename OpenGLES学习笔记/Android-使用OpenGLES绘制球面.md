从现在开始，我们不用Native层的方法来使用OpenGLES了。经过前面的介绍，Native层该怎么使用，应该来说都比较熟悉了。
我们从现在开始，使用Android封装的GLES20 接口，在Java层使用OpenGLES。不过对于性能要求比较高的绘制过程，还是应该交给Native层做处理，比如做瘦脸、美白、磨皮等算法运算量比较大的滤镜。
好了，废话不多说了。关于绘制球面，我们首先需要了解的是，这么去构建一个球面纹理。要构建球面，我们首先需要知道的顶点和纹理需要怎么构建。球面纹理一般情况下是通过将球面拆分成多个平面，然后使用三角形对平面进行构建。当拆分成的平面越多，则约像一个球。球面的顶点一般都是相互连接起来的，如下图所示：
![球面顶点.png](http://upload-images.jianshu.io/upload_images/2103804-8a729199484a4ef6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

既然知道了球面顶点的连接方式，那么该如何计算球面顶点的位置？
如下图所示：
![三维坐标公式.png](http://upload-images.jianshu.io/upload_images/2103804-dfdb1173ec58ccd8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
根据三角形公式可以得到：
x0 = R * cos(a) * sin(b);
y0 = R * sin(a);
z0 = R * cos(a) * cos(b);
我们可以由此得到一个球某个位置的顶点所在的位置。那么我们可以通过将球面拆分成许许多多的按照一定值的夹角的顶点，相互连接起来，就构成了我们所需要的球。计算顶点方法如下：
```
    for (double vAngle = 0; vAngle < Math.PI; vAngle = vAngle + angleSpan) { // vertical
            for (double hAngle = 0; hAngle < 2 * Math.PI; hAngle = hAngle + angleSpan) { // horizontal

                float x0 = (float) (radius * Math.sin(vAngle) * Math.cos(hAngle));
                float y0 = (float) (radius * Math.sin(vAngle) * Math.sin(hAngle));
                float z0 = (float) (radius * Math.cos((vAngle)));

                float x1 = (float) (radius * Math.sin(vAngle) * Math.cos(hAngle + angleSpan));
                float y1 = (float) (radius * Math.sin(vAngle) * Math.sin(hAngle + angleSpan));
                float z1 = (float) (radius * Math.cos(vAngle));

                float x2 = (float) (radius * Math.sin(vAngle + angleSpan) * Math.cos(hAngle + angleSpan));
                float y2 = (float) (radius * Math.sin(vAngle + angleSpan) * Math.sin(hAngle + angleSpan));
                float z2 = (float) (radius * Math.cos(vAngle + angleSpan));

                float x3 = (float) (radius * Math.sin(vAngle + angleSpan) * Math.cos(hAngle));
                float y3 = (float) (radius * Math.sin(vAngle + angleSpan) * Math.sin(hAngle));
                float z3 = (float) (radius * Math.cos(vAngle + angleSpan));

                vertex.add(x1);
                vertex.add(y1);
                vertex.add(z1);

                vertex.add(x3);
                vertex.add(y3);
                vertex.add(z3);

                vertex.add(x0);
                vertex.add(y0);
                vertex.add(z0);

                vertex.add(x1);
                vertex.add(y1);
                vertex.add(z1);

                vertex.add(x2);
                vertex.add(y2);
                vertex.add(z2);

                vertex.add(x3);
                vertex.add(y3);
                vertex.add(z3);
            }
        }
```
上面的方式就是通过横向和纵向按照一定的角度分割成多个顶点，将所有的顶点相互连接成一小块平面后，拼接起来就得到了我们所需要的球面。
好了，我们来看看具体实现吧：
自定义GLSurfaceView：
```
public class SphereSurfaceView extends GLSurfaceView {

    private SphereRender mSphereRender;

    public SphereSurfaceView(Context context) {
        super(context);
        init(context);
    }

    public SphereSurfaceView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init(context);
    }

    private void init(Context context) {
        mSphereRender = new SphereRender(context);
        setEGLContextClientVersion(2);
        setRenderer(mSphereRender);
        setRenderMode(GLSurfaceView.RENDERMODE_CONTINUOUSLY);
        setOnTouchListener(new OnTouchListener() {

            public boolean onTouch(View v, MotionEvent event) {
                switch (event.getAction()) {
                    case MotionEvent.ACTION_DOWN://检测到点击事件时
                        mSphereRender.rotate(20f, 0, 1, 0); //绕y轴旋转
                }
                return true;
            }
        });
    }
    
}
```
点击GLSurfaceView时，球面会绕Y轴旋转20度。
我们来看看Render的实现：
```
public class SphereRender implements GLSurfaceView.Renderer {

    // 视图矩阵
    private float[] mViewMatrix = new float[16];
    // 模型矩阵
    private float[] mModelMatrix = new float[16];
    // 投影矩阵
    private float[] mProjectionMatrix = new float[16];
    // 总变换矩阵
    private float[] mMVPMatrix = new float[16];

    private SphereFilter mSphereFilter;

    private Context mContext;

    public SphereRender(Context context) {
        mContext = context;
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        //设置屏幕背景色RGBA
        GLES20.glClearColor(0.5f, 0.5f, 0.5f, 1.0f);
        //打开深度检测
        GLES20.glEnable(GLES20.GL_DEPTH_TEST);
        //打开背面剪裁
        GLES20.glEnable(GLES20.GL_CULL_FACE);
        // 创建球面绘制实体
        mSphereFilter = new SphereFilter();
        // 初始化矩阵
        initMatrix();
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0, 0, width, height);
        float ratio = (float) width / height;
        // 调用此方法计算产生透视投影矩阵
        Matrix.frustumM(mProjectionMatrix, 0, -ratio, ratio, -1, 1, 20, 100);
        // 调用此方法产生摄像机9参数位置矩阵
        Matrix.setLookAtM(mViewMatrix, 0,
                0f, 0f, 30f,    // 相机位置
                0f, 0f, 0f,     // 目标位置
                0f, 1.0f, 0.0f  // 相机朝向
        );
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        //清除深度缓冲与颜色缓冲
        GLES20.glClear( GLES20.GL_DEPTH_BUFFER_BIT | GLES20.GL_COLOR_BUFFER_BIT);
        calculateMatrix();
        mSphereFilter.setMVPMatrix(mMVPMatrix);
        mSphereFilter.drawSphere();
    }

    /**
     * 初始化矩阵
     */
    private void initMatrix() {
        Matrix.setIdentityM(mViewMatrix, 0);
        Matrix.setIdentityM(mModelMatrix, 0);
        Matrix.setIdentityM(mProjectionMatrix, 0);
        Matrix.setIdentityM(mMVPMatrix, 0);
    }

    /**
     * 计算总变换矩阵
     */
    private void calculateMatrix() {
        Matrix.multiplyMM(mMVPMatrix, 0, mViewMatrix, 0, mModelMatrix, 0);
        Matrix.multiplyMM(mMVPMatrix, 0, mProjectionMatrix, 0, mMVPMatrix, 0);
    }

    /**
     * 物体旋转
     * @param angle
     * @param x
     * @param y
     * @param z
     */
    public void rotate(float angle, float x, float y, float z) {
        Matrix.rotateM(mModelMatrix, 0, angle, x, y, z);
    }
}
```
Render主要是启用了深度检测，然后设置相应的视图矩阵、投影矩阵和模型矩阵等，在绘制时计算出综合变换矩阵传递给球面绘制实体，然后绘制球面。球面绘制实体的实现如下：
```
public class SphereFilter {

    private static final String VERTEX_SHADER =
            "uniform mat4 u_Matrix;//最终的变换矩阵\n" +
            "attribute vec4 a_Position;//顶点位置\n" +
            "varying vec4 vPosition;//用于传递给片元着色器的顶点位置\n" +
            "void main() {\n" +
            "    gl_Position = u_Matrix * a_Position;\n" +
            "    vPosition = a_Position;\n" +
            "}";

    private static final String FRAGMENT_SHADER =
            "precision mediump float;\n" +
            "varying vec4 vPosition;\n" +
            "void main() {\n" +
            "    float uR = 0.6;//球的半径\n" +
            "    vec4 color;\n" +
            "    float n = 8.0;//分为n层n列n行\n" +
            "    float span = 2.0*uR/n;//正方形长度\n" +
            "    //计算行列层数\n" +
            "    int i = int((vPosition.x + uR)/span);//行数\n" +
            "    int j = int((vPosition.y + uR)/span);//层数\n" +
            "    int k = int((vPosition.z + uR)/span);//列数\n" +
            "    int colorType = int(mod(float(i+j+k),2.0));\n" +
            "    if(colorType == 1) {//奇数时为绿色\n" +
            "        color = vec4(0.2,1.0,0.129,0);\n" +
            "    } else {//偶数时为白色\n" +
            "        color = vec4(1.0,1.0,1.0,0);//白色\n" +
            "    }\n" +
            "    //将计算出的颜色给此片元\n" +
            "    gl_FragColor = color;\n" +
            "}";

    private float radius = 1.0f; // 球的半径
    final double angleSpan = Math.PI / 90f; // 将球进行单位切分的角度
    private FloatBuffer mVertexBuffer;// 顶点坐标
    int mVertexCount = 0;// 顶点个数，先初始化为0

    // float类型的字节数
    private static final int BYTES_PER_FLOAT = 4;

    // 数组中每个顶点的坐标数
    private static final int COORDS_PER_VERTEX = 3;

    private int mProgramHandle;
    private int muMatrixHandle;
    private int maPositionHandle;

    private float[] mMVPMatrix = new float[16];


    public SphereFilter() {
        initSphereVertex();
        createProgram();
        Matrix.setIdentityM(mMVPMatrix, 0);
    }

    /**
     * 计算球面顶点
     */
    public void initSphereVertex() {
        ArrayList<Float> vertex = new ArrayList<Float>();

        for (double vAngle = 0; vAngle < Math.PI; vAngle = vAngle + angleSpan) { // vertical
            for (double hAngle = 0; hAngle < 2 * Math.PI; hAngle = hAngle + angleSpan) { // horizontal

                float x0 = (float) (radius * Math.sin(vAngle) * Math.cos(hAngle));
                float y0 = (float) (radius * Math.sin(vAngle) * Math.sin(hAngle));
                float z0 = (float) (radius * Math.cos((vAngle)));

                float x1 = (float) (radius * Math.sin(vAngle) * Math.cos(hAngle + angleSpan));
                float y1 = (float) (radius * Math.sin(vAngle) * Math.sin(hAngle + angleSpan));
                float z1 = (float) (radius * Math.cos(vAngle));

                float x2 = (float) (radius * Math.sin(vAngle + angleSpan) * Math.cos(hAngle + angleSpan));
                float y2 = (float) (radius * Math.sin(vAngle + angleSpan) * Math.sin(hAngle + angleSpan));
                float z2 = (float) (radius * Math.cos(vAngle + angleSpan));

                float x3 = (float) (radius * Math.sin(vAngle + angleSpan) * Math.cos(hAngle));
                float y3 = (float) (radius * Math.sin(vAngle + angleSpan) * Math.sin(hAngle));
                float z3 = (float) (radius * Math.cos(vAngle + angleSpan));

                vertex.add(x1);
                vertex.add(y1);
                vertex.add(z1);

                vertex.add(x3);
                vertex.add(y3);
                vertex.add(z3);

                vertex.add(x0);
                vertex.add(y0);
                vertex.add(z0);

                vertex.add(x1);
                vertex.add(y1);
                vertex.add(z1);

                vertex.add(x2);
                vertex.add(y2);
                vertex.add(z2);

                vertex.add(x3);
                vertex.add(y3);
                vertex.add(z3);
            }
        }

        mVertexCount = vertex.size() / COORDS_PER_VERTEX;
        float vertices[] = new float[vertex.size()];
        for (int i = 0; i < vertex.size(); i++) {
            vertices[i] = vertex.get(i);
        }
        mVertexBuffer = GlUtil.createFloatBuffer(vertices);
    }

    /**
     * 创建Program
     */
    private void createProgram() {
        mProgramHandle = GlUtil.createProgram(VERTEX_SHADER, FRAGMENT_SHADER);
        maPositionHandle = GLES20.glGetAttribLocation(mProgramHandle, "a_Position");
        muMatrixHandle = GLES20.glGetUniformLocation(mProgramHandle, "u_Matrix");
    }

    /**
     * 绘制球面
     */
    public void drawSphere() {
        GLES20.glUseProgram(mProgramHandle);
        GLES20.glVertexAttribPointer(maPositionHandle, COORDS_PER_VERTEX,
                GLES20.GL_FLOAT, false, 0, mVertexBuffer);
        GLES20.glEnableVertexAttribArray(maPositionHandle);
        GLES20.glUniformMatrix4fv(muMatrixHandle, 1, false, mMVPMatrix, 0);
        GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, mVertexCount);
        GLES20.glDisableVertexAttribArray(maPositionHandle);
        GLES20.glUseProgram(0);
    }

    /**
     * 设置最终的变换矩阵
     * @param matrix
     */
    public void setMVPMatrix(float[] matrix) {
        mMVPMatrix = matrix;
    }
}
```
实现过程主要是创建Program、绑定对应的Attribute属性、设置总变换矩阵、绘制球面等。下面是工具，用于创建Program、创建mipmap纹理等方法的封装：
```
public class GlUtil {

    public static final String TAG = "GlUtil";

    // 从初始化失败
    public static final int GL_NOT_INIT = -1;

    // 单位矩阵
    public static final float[] IDENTITY_MATRIX;
    static {
        IDENTITY_MATRIX = new float[16];
        Matrix.setIdentityM(IDENTITY_MATRIX, 0);
    }

    private static final int SIZEOF_FLOAT = 4;

    private GlUtil() {}

    /**
     * 创建program
     * @param vertexSource
     * @param fragmentSource
     * @return
     */
    public static int createProgram(String vertexSource, String fragmentSource) {
        int vertexShader = loadShader(GLES30.GL_VERTEX_SHADER, vertexSource);
        if (vertexShader == 0) {
            return 0;
        }
        int pixelShader = loadShader(GLES30.GL_FRAGMENT_SHADER, fragmentSource);
        if (pixelShader == 0) {
            return 0;
        }

        int program = GLES30.glCreateProgram();
        checkGlError("glCreateProgram");
        if (program == 0) {
            Log.e(TAG, "Could not create program");
        }
        GLES30.glAttachShader(program, vertexShader);
        checkGlError("glAttachShader");
        GLES30.glAttachShader(program, pixelShader);
        checkGlError("glAttachShader");
        GLES30.glLinkProgram(program);
        int[] linkStatus = new int[1];
        GLES30.glGetProgramiv(program, GLES30.GL_LINK_STATUS, linkStatus, 0);
        if (linkStatus[0] != GLES30.GL_TRUE) {
            Log.e(TAG, "Could not link program: ");
            Log.e(TAG, GLES30.glGetProgramInfoLog(program));
            GLES30.glDeleteProgram(program);
            program = 0;
        }
        return program;
    }

    /**
     * 加载Shader
     * @param shaderType
     * @param source
     * @return
     */
    public static int loadShader(int shaderType, String source) {
        int shader = GLES30.glCreateShader(shaderType);
        checkGlError("glCreateShader type=" + shaderType);
        GLES30.glShaderSource(shader, source);
        GLES30.glCompileShader(shader);
        int[] compiled = new int[1];
        GLES30.glGetShaderiv(shader, GLES30.GL_COMPILE_STATUS, compiled, 0);
        if (compiled[0] == 0) {
            Log.e(TAG, "Could not compile shader " + shaderType + ":");
            Log.e(TAG, " " + GLES30.glGetShaderInfoLog(shader));
            GLES30.glDeleteShader(shader);
            shader = 0;
        }
        return shader;
    }

    /**
     * 检查是否出错
     * @param op
     */
    public static void checkGlError(String op) {
        int error = GLES30.glGetError();
        if (error != GLES30.GL_NO_ERROR) {
            String msg = op + ": glError 0x" + Integer.toHexString(error);
            Log.e(TAG, msg);
            throw new RuntimeException(msg);
        }
    }

    /**
     * 创建FloatBuffer
     * @param coords
     * @return
     */
    public static FloatBuffer createFloatBuffer(float[] coords) {
        ByteBuffer bb = ByteBuffer.allocateDirect(coords.length * SIZEOF_FLOAT);
        bb.order(ByteOrder.nativeOrder());
        FloatBuffer fb = bb.asFloatBuffer();
        fb.put(coords);
        fb.position(0);
        return fb;
    }

    /**
     * 创建FloatBuffer
     * @param data
     * @return
     */
    public static FloatBuffer createFloatBuffer(ArrayList<Float> data) {
        float[] coords = new float[data.size()];
        for (int i = 0; i < coords.length; i++){
            coords[i] = data.get(i);
        }
        return createFloatBuffer(coords);
    }

    /**
     * 创建Texture对象
     * @param textureType
     * @return
     */
    public static int createTextureObject(int textureType) {
        int[] textures = new int[1];
        GLES30.glGenTextures(1, textures, 0);
        GlUtil.checkGlError("glGenTextures");
        int textureId = textures[0];
        GLES30.glBindTexture(textureType, textureId);
        GlUtil.checkGlError("glBindTexture " + textureId);
        GLES30.glTexParameterf(textureType, GLES30.GL_TEXTURE_MIN_FILTER, GLES30.GL_NEAREST);
        GLES30.glTexParameterf(textureType, GLES30.GL_TEXTURE_MAG_FILTER, GLES30.GL_LINEAR);
        GLES30.glTexParameterf(textureType, GLES30.GL_TEXTURE_WRAP_S, GLES30.GL_CLAMP_TO_EDGE);
        GLES30.glTexParameterf(textureType, GLES30.GL_TEXTURE_WRAP_T, GLES30.GL_CLAMP_TO_EDGE);
        GlUtil.checkGlError("glTexParameter");
        return textureId;
    }

    /**
     * 创建Sampler2D的Framebuffer 和 Texture
     * @param frameBuffer
     * @param frameBufferTex
     * @param width
     * @param height
     */
    public static void createSampler2DFrameBuff(int[] frameBuffer, int[] frameBufferTex,
                                                int width, int height) {
        GLES30.glGenFramebuffers(1, frameBuffer, 0);
        GLES30.glGenTextures(1, frameBufferTex, 0);
        GLES30.glBindTexture(GLES30.GL_TEXTURE_2D, frameBufferTex[0]);
        GLES30.glTexImage2D(GLES30.GL_TEXTURE_2D, 0, GLES30.GL_RGBA, width, height, 0,
                GLES30.GL_RGBA, GLES30.GL_UNSIGNED_BYTE, null);
        GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                GLES30.GL_TEXTURE_MAG_FILTER, GLES30.GL_LINEAR);
        GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                GLES30.GL_TEXTURE_MIN_FILTER, GLES30.GL_LINEAR);
        GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                GLES30.GL_TEXTURE_WRAP_S, GLES30.GL_CLAMP_TO_EDGE);
        GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                GLES30.GL_TEXTURE_WRAP_T, GLES30.GL_CLAMP_TO_EDGE);
        GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, frameBuffer[0]);
        GLES30.glFramebufferTexture2D(GLES30.GL_FRAMEBUFFER, GLES30.GL_COLOR_ATTACHMENT0,
                GLES30.GL_TEXTURE_2D, frameBufferTex[0], 0);
        GLES30.glBindTexture(GLES30.GL_TEXTURE_2D, 0);
        GLES30.glBindFramebuffer(GLES30.GL_FRAMEBUFFER, 0);
        checkGlError("createCamFrameBuff");
    }

    /**
     * 加载mipmap纹理
     * @param bitmap bitmap图片
     * @return
     */
    public static int createTexture(Bitmap bitmap) {
        int[] texture = new int[1];
        if (bitmap != null && !bitmap.isRecycled()) {
            //生成纹理
            GLES30.glGenTextures(1, texture, 0);
            checkGlError("glGenTexture");
            //生成纹理
            GLES30.glBindTexture(GLES30.GL_TEXTURE_2D, texture[0]);
            //设置缩小过滤为使用纹理中坐标最接近的一个像素的颜色作为需要绘制的像素颜色
            GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                    GLES30.GL_TEXTURE_MIN_FILTER,GLES30.GL_NEAREST);
            //设置放大过滤为使用纹理中坐标最接近的若干个颜色，通过加权平均算法得到需要绘制的像素颜色
            GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                    GLES30.GL_TEXTURE_MAG_FILTER,GLES30.GL_LINEAR);
            //设置环绕方向S，截取纹理坐标到[1/2n,1-1/2n]。将导致永远不会与border融合
            GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                    GLES30.GL_TEXTURE_WRAP_S,GLES30.GL_CLAMP_TO_EDGE);
            //设置环绕方向T，截取纹理坐标到[1/2n,1-1/2n]。将导致永远不会与border融合
            GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                    GLES30.GL_TEXTURE_WRAP_T,GLES30.GL_CLAMP_TO_EDGE);
            //根据以上指定的参数，生成一个2D纹理
            GLUtils.texImage2D(GLES30.GL_TEXTURE_2D, 0, bitmap, 0);
            return texture[0];
        }
        return 0;
    }

    /**
     * 创建OES类型的Texture
     * @return
     */
    public static int createOESTexture() {
        int[] texture = new int[1];

        GLES20.glGenTextures(1, texture, 0);
        GLES20.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, texture[0]);
        GLES20.glTexParameterf(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
                GL10.GL_TEXTURE_MIN_FILTER, GL10.GL_LINEAR);
        GLES20.glTexParameterf(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
                GL10.GL_TEXTURE_MAG_FILTER, GL10.GL_LINEAR);
        GLES20.glTexParameteri(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
                GL10.GL_TEXTURE_WRAP_S, GL10.GL_CLAMP_TO_EDGE);
        GLES20.glTexParameteri(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
                GL10.GL_TEXTURE_WRAP_T, GL10.GL_CLAMP_TO_EDGE);

        return texture[0];
    }

    /**
     * 加载mipmap纹理
     * @param context
     * @param name
     * @return
     */
    public static int loadMipmapTextureFromAssets(Context context, String name) {
        int[] textureHandle = new int[1];
        GLES30.glGenTextures(1, textureHandle, 0);
        if (textureHandle[0] != 0) {
            Bitmap bitmap = getImageFromAssetsFile(context, name);
            GLES30.glBindTexture(GLES30.GL_TEXTURE_2D, textureHandle[0]);

            GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                    GLES30.GL_TEXTURE_MAG_FILTER, GLES30.GL_LINEAR);
            GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                    GLES30.GL_TEXTURE_MIN_FILTER, GLES30.GL_LINEAR);
            GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                    GLES30.GL_TEXTURE_WRAP_S, GLES30.GL_CLAMP_TO_EDGE);
            GLES30.glTexParameterf(GLES30.GL_TEXTURE_2D,
                    GLES30.GL_TEXTURE_WRAP_T, GLES30.GL_CLAMP_TO_EDGE);
            GLUtils.texImage2D(GLES30.GL_TEXTURE_2D, 0, bitmap, 0);
            bitmap.recycle();
        }
        if (textureHandle[0] == 0) {
            throw new RuntimeException("Error loading texture.");
        }

        return textureHandle[0];
    }

    /**
     * 加载Assets文件夹下的图片
     * @param context
     * @param fileName
     * @return
     */
    public static Bitmap getImageFromAssetsFile(Context context, String fileName) {
        Bitmap bitmap = null;
        AssetManager manager = context.getResources().getAssets();
        try {
            InputStream is = manager.open(fileName);
            bitmap = BitmapFactory.decodeStream(is);
            is.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return bitmap;
    }
}
```
至此，我们就可以看到一个球面了：
![Screenshot_20170829-115027.png](http://upload-images.jianshu.io/upload_images/2103804-620348e97d601571.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
