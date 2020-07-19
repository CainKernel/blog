前面我们说到了天空盒，下面我们来看看粒子Particle是怎样实现的。我们在《Android 使用OpenGLES制作天空盒》（地址：
 http://www.jianshu.com/p/820581046d3c）的基础上实现粒子系统：
1、在SkyBoxRenderer类中添加ParticleFilter：
```
    @Override
    public void onSurfaceCreated(GL10 glUnused, EGLConfig config) {
        GLES20.glClearColor(0.0f, 0.0f, 0.0f, 0.0f);

        // 创建天空盒
        mSkyBoxFilter = new SkyBoxFilter(mContext);
        mSkyBoxFilter.createProgram();

        // 创建粒子系统
        mParticleFilter = new ParticleFilter(mContext);
        mParticleFilter.createProgram();
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
        if (mParticleFilter != null) {
            mParticleFilter.setViewSize(width, height);
        }
    }

    @Override    
    public void onDrawFrame(GL10 glUnused) {
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
        // 绘制天空盒
        drawSkybox();
        // 绘制粒子系统
        drawParticles();
    }
    /**
     * 绘制粒子特效
     */
    private void drawParticles() {
        // 混合
        GLES20.glEnable(GLES20.GL_BLEND);
        GLES20.glBlendFunc(GLES20.GL_ONE, GLES20.GL_ONE);
        if (mParticleFilter != null) {
            mParticleFilter.drawParticle();
        }
        GLES20.glDisable(GLES20.GL_BLEND);

    }
```
ParticleFilter类主要是通过计算位置、颜色、运动方向等实现粒子的随机运动， 实现如下：
```
public class ParticleFilter {

    // 最大粒子数
    private static final int MaxParticleCount = 1000;

    private Context mContext;
    private int mViewWidth;
    private int mViewHeight;

    // 创建粒子系统开始时间
    private long mGlobalStartTime;

    // GLSL句柄
    private int mProgramHandle;
    private int mMVPMatrixHandle;
    private int uTimeLocation;
    private int aPositionLocation;
    private int aColorLocation;
    private int aDirectionVectorLocation;
    private int aParticleStartTimeLocation;
    private int aOpenMouthLocation;

    private int mParticleTexture;

    private float[] mProjectionMatrix = new float[16];
    private float[] mViewMatrix = new float[16];
    private float[] mMVPMatrix = new float[16];


    private static final int POSITION_COMPONENT_COUNT = 3;
    private static final int COLOR_COMPONENT_COUNT = 3;
    private static final int VECTOR_COMPONENT_COUNT = 3;
    private static final int PARTICLE_START_TIME_COMPONENT_COUNT = 1;

    private static final int TOTAL_COMPONENT_COUNT =
            POSITION_COMPONENT_COUNT
                    + COLOR_COMPONENT_COUNT
                    + VECTOR_COMPONENT_COUNT
                    + PARTICLE_START_TIME_COMPONENT_COUNT;

    private static final int STRIDE = TOTAL_COMPONENT_COUNT * BYTES_PER_FLOAT;

    private float[] mParticles;
    private FloatBuffer mVertexBuffer;
    private int mCurrentParticleCount;
    private int mNextParticle;



    // ParticleShooter内容
    private Geometry.Point mPosition; // 起始位置
    private int mColor; // 颜色

    private float mAngleVariance;
    private float mSpeedVariance;

    private Random mRandom = new Random();

    // 旋转矩阵
    private float[] mRotationMatrix = new float[16];
    // 目的向量
    private float[] mDirectionVector = new float[4];
    // 最终的结果
    private float[] mResultVector = new float[4];


    public ParticleFilter(Context context) {
        mContext = context;
        mGlobalStartTime = System.nanoTime();

        mPosition = new Geometry.Point(0f, 0f, 0f);
        mDirectionVector[0] = 0.0f;
        mDirectionVector[1] = 0.5f;
        mDirectionVector[2] = 0.0f;
        mAngleVariance = 5f;
        mSpeedVariance = 1.0f;
        mColor = Color.rgb(255, 255, 255);

        // 创建粒子
        mParticles = new float[MaxParticleCount * TOTAL_COMPONENT_COUNT];
        mVertexBuffer = GlUtil.createFloatBuffer(mParticles);
    }

    /**
     * 创建Program
     */
    public void createProgram() {
        String vertexShader = ResourceUtil.readTextFileFromResource(mContext,
                R.raw.vertex_particle);
        String fragmentShader = ResourceUtil.readTextFileFromResource(mContext,
                R.raw.fragment_particle);
        mProgramHandle = GlUtil.createProgram(vertexShader, fragmentShader);

        mMVPMatrixHandle = GLES20.glGetUniformLocation(mProgramHandle, "uMVPMatrix");
        uTimeLocation = GLES20.glGetUniformLocation(mProgramHandle, "uTime");
        aOpenMouthLocation = GLES20.glGetUniformLocation(mProgramHandle, "a_OpenMouth");

        aPositionLocation = GLES20.glGetAttribLocation(mProgramHandle, "a_Position");
        aColorLocation = GLES20.glGetAttribLocation(mProgramHandle, "a_Color");
        aDirectionVectorLocation = GLES20.glGetAttribLocation(mProgramHandle, "a_DirectionVector");
        aParticleStartTimeLocation = GLES20.glGetAttribLocation(mProgramHandle, "a_ParticleStartTime");

        // 创建粒子的Texture
        mParticleTexture = TextureHelper.loadTexture(mContext, R.drawable.particle_texture);
    }

    /**
     * 更新视图大小
     * @param width
     * @param height
     */
    public void setViewSize(int width, int height) {
        mViewWidth = width;
        mViewHeight = height;
        // 设置透视矩阵
        float ratio = (float) width / (float) height;
        MatrixHelper.perspectiveM(mProjectionMatrix, 45, ratio, 1f, 300f);
    }

    /**
     * 绘制粒子
     */
    public void drawParticle() {
        GLES20.glUseProgram(mProgramHandle);
        float currentTime = (System.nanoTime() - mGlobalStartTime) / 1000000000f;
        GLES20.glUniform1f(uTimeLocation, currentTime);
        calculate(currentTime, 1);
        calculateMatirx();
        GLES20.glUniformMatrix4fv(mMVPMatrixHandle, 1, false, mMVPMatrix, 0);
        GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, mParticleTexture);
        bindData();
        GLES20.glDrawArrays(GLES20.GL_POINTS, 0, mCurrentParticleCount);

        GLES20.glUseProgram(0);
    }

    private void bindData() {
        int dataOffset = 0;
        // position
        mVertexBuffer.position(dataOffset);
        GLES20.glEnableVertexAttribArray(aPositionLocation);
        GLES20.glVertexAttribPointer(aPositionLocation, POSITION_COMPONENT_COUNT,
                GLES20.GL_FLOAT, false, STRIDE, mVertexBuffer);
        mVertexBuffer.position(0);
        dataOffset += POSITION_COMPONENT_COUNT;

        // color
        mVertexBuffer.position(dataOffset);
        GLES20.glEnableVertexAttribArray(aColorLocation);
        GLES20.glVertexAttribPointer(aColorLocation, COLOR_COMPONENT_COUNT,
                GLES20.GL_FLOAT, false, STRIDE, mVertexBuffer);
        mVertexBuffer.position(0);
        dataOffset += COLOR_COMPONENT_COUNT;

        // direction
        mVertexBuffer.position(dataOffset);
        GLES20.glEnableVertexAttribArray(aDirectionVectorLocation);
        GLES20.glVertexAttribPointer(aDirectionVectorLocation, VECTOR_COMPONENT_COUNT,
                GLES20.GL_FLOAT, false, STRIDE, mVertexBuffer);
        mVertexBuffer.position(0);
        dataOffset += VECTOR_COMPONENT_COUNT;

        // startTime
        mVertexBuffer.position(dataOffset);
        GLES20.glEnableVertexAttribArray(aParticleStartTimeLocation);
        GLES20.glVertexAttribPointer(aParticleStartTimeLocation,
                PARTICLE_START_TIME_COMPONENT_COUNT,
                GLES20.GL_FLOAT, false, STRIDE, mVertexBuffer);
        mVertexBuffer.position(0);
    }

    /**
     * 计算粒子
     * @param time
     * @param count
     */
    private void calculate(float time, int count) {
        for (int i = 0; i < count; i++) {
            // 随机产生(-1.0 ~ 1.0)之间的起始位置
            float startX = mRandom.nextFloat() * 2 - 1.0f;
            float startY = mRandom.nextFloat() * 2 - 1.0f;
            mPosition = new Geometry.Point(startX, startY, 0);

            // 设置欧拉角
            Matrix.setRotateEulerM(mRotationMatrix, 0,
                    (mRandom.nextFloat() - 0.5f) * mAngleVariance,
                    (mRandom.nextFloat() - 0.5f) * mAngleVariance,
                    (mRandom.nextFloat() - 0.5f) * mAngleVariance);

            // 向量矩阵乘法
            Matrix.multiplyMV(mResultVector, 0, mRotationMatrix, 0, mDirectionVector, 0);

            // 调整速度
            float speedAdjustment = 1f + mRandom.nextFloat() * mSpeedVariance;

            // 计算最终的方向向量
            Geometry.Vector direction = new Geometry.Vector(
                    mResultVector[0] * speedAdjustment,
                    mResultVector[1] * speedAdjustment,
                    mResultVector[2] * speedAdjustment);

            addParticle(mPosition, mColor, direction, time);
        }
    }

    /**
     * 添加粒子
     * @param position
     * @param color
     * @param direction
     * @param startTime
     */
    private void addParticle(Geometry.Point position, int color,
                             Geometry.Vector direction, float startTime) {
        final int particleOffset = mNextParticle * TOTAL_COMPONENT_COUNT;
        int currentOffset = particleOffset;
        mNextParticle++;

        if (mCurrentParticleCount < MaxParticleCount) {
            mCurrentParticleCount++;
        }

        if (mNextParticle == MaxParticleCount) {
            mNextParticle = 0;
        }

        // 原始位置
        mParticles[currentOffset++] = position.x;
        mParticles[currentOffset++] = position.y;
        mParticles[currentOffset++] = position.z;

        // 颜色
        mParticles[currentOffset++] = Color.red(color) / 255f;
        mParticles[currentOffset++] = Color.green(color) / 255f;
        mParticles[currentOffset++] = Color.blue(color) / 255f;

        // 目的位置
        mParticles[currentOffset++] = direction.x;
        mParticles[currentOffset++] = direction.y;
        mParticles[currentOffset++] = direction.z;

        mParticles[currentOffset++] = startTime;

        mVertexBuffer.position(particleOffset);
        mVertexBuffer.put(mParticles, particleOffset, TOTAL_COMPONENT_COUNT);
        mVertexBuffer.position(0);
    }

    /**
     * 计算总变换
     */
    private void calculateMatirx() {
        // 计算综合矩阵
        Matrix.setIdentityM(mViewMatrix, 0);
        Matrix.translateM(mViewMatrix, 0, 0f, 0f, -5f);
        Matrix.multiplyMM(mMVPMatrix, 0, mProjectionMatrix, 0, mViewMatrix, 0);
    }

    /**
     * 设置点击的位置
     * @param x
     * @param y
     * @param z
     */
    public void setDirection(float x, float y, float z) {
        mDirectionVector[0] = x;
        mDirectionVector[1] = y;
        mDirectionVector[2] = z;
    }

}
```
GLSL的代码如下：
VertexShader:
```
uniform mat4 uMVPMatrix;
uniform float uTime;
uniform int a_OpenMouth;

attribute vec3 a_Position;  
attribute vec3 a_Color;
attribute vec3 a_DirectionVector;
attribute float a_ParticleStartTime;


varying vec3 v_Color;
varying float v_ElapsedTime;

void main()
{
    // 颜色
    v_Color = a_Color;
    // 时间经过的
    v_ElapsedTime = uTime - a_ParticleStartTime;
    // 重力加速度
    float gravityFactor = v_ElapsedTime * v_ElapsedTime / 8.0;
    // 当前位置
    vec3 currentPosition = a_Position + (a_DirectionVector * v_ElapsedTime);
    // 如果张开嘴巴，则将当前位置运动到嘴巴中心点
    if (a_OpenMouth == 1) {
        currentPosition = a_Position + ((a_DirectionVector - a_Position) * v_ElapsedTime);
    } else {
        currentPosition.y -= gravityFactor;
    }
    gl_Position = uMVPMatrix * vec4(currentPosition, 1.0);
    gl_PointSize = 25.0;
}
```
FragmentShader:
```
precision mediump float; 

uniform sampler2D u_TextureUnit;

varying vec3 v_Color;
varying float v_ElapsedTime;
      	 							    	   								
void main()                    		
{
    gl_FragColor = vec4(v_Color / v_ElapsedTime, 1.0)
                 * texture2D(u_TextureUnit, gl_PointCoord);
}
```
实现的效果如下：

![Screenshot_20170831-160500.png](http://upload-images.jianshu.io/upload_images/2103804-592f36b3c047d9db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

详情请参考：
https://github.com/CainKernel/SkyBox
