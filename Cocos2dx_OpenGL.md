# Cocos2dx加载默认着色器流程
1. 获取GLProgramCache实例(单例模式)
2. 实例一个GLProgram对象(着色器对象)
3. 根据配置的顶点着色器文件和片段着色器文件对GLProgram对象进行初始化
4. 执行GLProgram对象的链接函数(编译链接着色器,并设置顶点属性位置)
    * 如何将顶点数据传到顶点着色器。
      - 通过glBindAttribLocation函数来指定位置,在顶点着色器用in来接收,变量名称一致
      - 通过在顶点着色器中使用layout关键字指定位置。即layout (location = 0) in vec3 aPos;
      通过glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0)函数的第一个参数与上面两种指定位置的位置一致,来达到传值的目的
5. 保存着色器中uniform类型参数的位置。(uniform在着色器中是只读的,只能通过OpenGL向其传值)
     * 如何将uniform数据传到着色器。
       - 在着色器中定义uniform类型的变量。在OpenGL通过glGetUniformLocation(shaderProgram, "ourColor")函数来获取定义的uniform类型的变量的位置。第一个
         参数着色器对象,第二个参数是着色器中定义uniform类型的变量名。通过一系列glUniform*(pos,value)将uniform数据传到着色器。
         
# 节点使用着色器
   不同的节点可以使用同一个着色器对象。但是可以使用不同顶点属性数据和uniform数据。通过使用GLProgramState对象来实现上述操作。GLProgramState对象中包含
   一个着色器对象和顶点属性数据和uniform数据。每个节点包含一个GLProgramState对象。节点通过对GLProgramState对象设置不同的顶点属性数据和uniform数据来实现
   上述操作。
   
   
# 如何在cocos2dx中使用着色器。
  1. 从GLProgramCache中获取一个着色器。
  2. 设置当前节点的着色器为刚获取的着色器。
  3. 重写节点的visit和draw函数。在visit函数中生成一个绘制命令加到渲染队列。在绘制命令中指定绘制的回调为draw函数。
  4. 在draw函数中获取着色器。编写顶点着色器和片段找色器文件。编译链接着色器文件,并设置着色器顶点属性位置和unifrom类型变量的位置。
  5. 将定义的顶点数据和unifrom类型传到着色器。
  6. 执行渲染流程。
  
  ```cpp
  Scene* HelloWorld::createScene()
{
    // 'scene' is an autorelease object
    auto scene = Scene::create();

    // 'layer' is an autorelease object
    auto layer = HelloWorld::create();

    // add layer as a child to scene
    scene->addChild(layer);

    // return the scene
    return scene;
}

// on "init" you need to initialize your instance
bool HelloWorld::init()
{
    //////////////////////////////
    // 1. super init first
    if ( !Layer::init() )
    {
        return false;
    }

    /*
    在OpenGL中，GLSL的shader使用的流程与C语言相似，每个shader类似一个C模块，首先需要单独编译（compile），
    然后一组编译好的shader连接（link）成一个完整程序。
    */
    auto program = CCGLProgram::createWithFilenames("shader/vert.vsh", "shader/frag.fsh");
    program->link();
    program->updateUniforms();
    this->setGLProgram(program);

    /*
    使用VBO和VAO的步骤都差不多，步骤如下：
    1 glGenXXX
    2 glBindXXX
    */

    // 创建和绑定vao
    glGenVertexArrays(1, &_vao); 
    glBindVertexArray(_vao);

    // 创建和绑定vbo
    glGenBuffers(1, &_vertVBO);// 生成VBO
    glBindBuffer(GL_ARRAY_BUFFER, _vertVBO); // 关联到当前的VAO

    auto size = Director::getInstance()->getVisibleSize();
    float vertercies[] = {// 三角形顶点位置
        0, 0, // 第1个点坐标
        size.width, 0, // 第2个点坐标
        size.width / 2, size.height // 第3个点坐标
    };

    // 给VBO设置数据
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertercies), vertercies, GL_STATIC_DRAW);

    // 获得变量a_position在内存中的位置
    GLuint positionLocation = glGetAttribLocation(program->getProgram(), "a_position");
    glEnableVertexAttribArray(positionLocation);
    // 提交包含数据的数组指针
    glVertexAttribPointer(GLProgram::VERTEX_ATTRIB_POSITION, 2, GL_FLOAT, GL_FALSE, 0, (GLvoid*)0);

    // 设置颜色
    float color[] = {// 三角形顶点颜色RGBA
        0, 1, 0, 1,
        1, 0, 0, 1,
        0, 0, 1, 1
    };
    glGenBuffers(1, &_colorVBO);
    glBindBuffer(GL_ARRAY_BUFFER, _colorVBO);
    glBufferData(GL_ARRAY_BUFFER, sizeof(color), color, GL_STATIC_DRAW);

    // 获得变量a_color在内存中的位置
    GLuint colorLocation = glGetAttribLocation(program->getProgram(), "a_color");
    glEnableVertexAttribArray(colorLocation);
    glVertexAttribPointer(GLProgram::VERTEX_ATTRIB_COLOR, 4, GL_FLOAT, GL_FALSE, 0, (GLvoid*)0);

    glBindVertexArray(0);
    glBindBuffer(GL_ARRAY_BUFFER, 0);

    return true;
}

void HelloWorld::visit(cocos2d::Renderer *renderer, const Mat4 &transform, uint32_t parentFlags)
{
    Layer::draw(renderer, transform, parentFlags);

    _customCommand.init(_globalZOrder);
    _customCommand.func = CC_CALLBACK_0(HelloWorld::onDraw, this);
    renderer->addCommand(&_customCommand);
}

void HelloWorld::onDraw()
{
    auto glProgram = getGLProgram();
    glProgram->use();
    glProgram->setUniformsForBuiltins();

    /*
    VAO里的VBOs都设置好了以后，在绘制的地方只需要设置当前绑定的VAO是哪个，
    就能按照初始化的VAO来绘制，即调用glDrawArrays
    */

    // 设置当前绑定的VAO
    glBindVertexArray(_vao);

    // 绘制三角形
    glDrawArrays(GL_TRIANGLES, 0, 3);

    // 解绑当前VAO，但并不释放
    glBindVertexArray(0);

    CC_INCREMENT_GL_DRAWN_BATCHES_AND_VERTICES(1, 3);
    CHECK_GL_ERROR_DEBUG();
}
  ```
  



# cocos2dx的新的渲染方式
  1. 生成绘制命令
     - 在UI元素内部，将不再进行具体的绘制工作，而是生成一个绘制命令，该绘制命令会携带所有绘制需要用到的属性信息，该命令会被压入绘制桟
  2. 绘制命令排序 
     - 绘制命令cmd将被添加至绘制命令桟RenderQueue。向RenderQueue压入数据时，会基于UI的GlobalOrder进行分组存储。
  3. 执行绘制命令
     - 采用批绘制的方式以加速绘制效率。对于相邻的cmd，如果它们使用相同的纹理、着色器等绘制特征，这只调用一次OpenGL ES绘制命令
  
