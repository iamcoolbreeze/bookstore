#项目开发流程
(目前还没有补充完整。。)
##1 前台模块

###1.1用户模块

  * 相关类或者页面及其主要方法功能

    * domain

        * User：封装用户数据

    * dao

        * UserDao：操作数据库

    * service

        * UserService：包括ajax和提交表单之后的用户名是否重复的校验，用户激活等

        * UserException：自定义用户异常类

    * web.servlet

        * UserServlet：继承于BaseServlet，封装数据，发送邮件等

    * jsp页面：

        * regist.jsp：用户注册页面

        * msg.jsp：信息回显页面



  * 用户注册：流程及各层的内容

    1. regist.jsp：

        * 注册页面，发送表单数据到UserServlet

        * js校验：

            * 简单校验：各表单项目不能为空，用户名太长，用户名只能为数字字母下划线组合(？如果不是会出现什么问题？？)，密码不能太短，两次输入密码要一致，邮箱格式不对等等

            * ajax校验：校验用户名是否已经被注册，请求UserServlet#verifyUsername()

            * 验证码：防止注册机，请求UserServlet#verifyCode()，对于验证码的校验，因为本地js校验存在一些问题，比如session取验证码出现不一致，安全性的问题，则在服务端利用session进行验证

    2. servlet层：       

        * VerifyUsernameServlet#doPost()：响应ajax传递username参数调用service层的verifyUsername方法查询username是否已经被注册

        * VerifyCodeServlet#doGet()：发送验证码图片到regist.jsp页面中的，并把验证码regist_verifyCode放入session中

        * UserServlet#verifyUsername()：调用UserService#verifyUsername()，传递username参数

        * UserServlet#regist()：

            * 封装用户的表单数据到user对象中

            * 对用户名，密码，邮箱，验证码等进行基础校验，如果有错误，保存错误信息和表单数据转发到regist.jsp回显

                * 如果usernmae，password为空值(虽然有js校验，但是客户端可能禁用js)，需要防止空指针异常，每次应该先判断是否为空if(user.getUsername==null ..)

            * 如果校验通过，则调用service层的regist方法，传递user对象

    3. service层：

        * UserService#verifyUsername(String username)：调用dao层findByUsername方法，返回信息

        * UserService#regist(User user)

            * 做进一步的校验

                * 调用dao层findByUsername方法，用户名已经被注册了，抛出自定义异常UserException

            * 如果校验通过则调用dao层的addUser方法，传递user对象              

    4. dao层：

        * 通过用户名查找记录findByUsername(String username)

        * 添加用户addUser(User user)



  * 用户激活：流程及各层的内容

    1. Servlet层：

        * UserServlet#regist()

            * 如果用户的校验都通过了，则构建唯一的激活码(uuid+uuid)，封装到user对象中，并调用sendActivationCodeLinkMail方法发送激活码链接到用户邮箱

            * 如果发送激活邮箱失败了，提示用户可以点击重新发送邮箱请求

        * UserServlet#sendActivationCodeLinkMail()

            * 加载邮箱配置文件properties，读取smtp服务器地址，用户名，密码，发件人，邮件主题，内容等参数

            * 使用JavaMail工具给注册用户发送邮件

        * UserServlet#sendMailAgain()

            * 如果用户没有收到激活码链接，点击了重新发送，则请求此方法，可以重新调用发送邮件的方法来重新发送邮件

    2. 用户邮箱：点击链接发送请求到UserServelt#activate()

    3. Servlet层：

        * UserServlet#activate()：

            * 调用service层的activate方法，传递激活码参数activationcode

            * 如果成功或者，则把信息传递给msg.jsp来回显给用户

    4.service层：

        * UserService#activate(String activationcode)

            * 调用dao层的方法通过激活码来进行激活操作

    5.dao层：

        * UserDao#findByActivationCode(String acitivationcode)

            * 如果通过查询不到用户，则抛出自定义异常

            * 如果查询的用户已经激活了(state=true)，则抛出异常

            * 操作成功，则设置用户state为true



  * 用户登录    

    1. 页面login.jsp

        * 基础js校验：用户名，密码不能为空，如果有验证码，验证码不能为空

        * 登录次数限制：超过3次错误，需要填写验证码，使用session来保存登录次数

        * 发送用户名和密码到UserServlet#login()进行登录验证

    2. Servlet层：UserServelt#login()

        1. 封装表单数据到user对象

        2. 调用UserService#login()，传递user对象

        3. 捕获相应的UserException，返回错误信息回显到login.jsp

            * 如果是用户名或者密码错误或者为激活，返回信息回显

            * 如果错误次数超过3次，则login.jsp需要输入验证码，使用session保存错误次数

            * 如果sessiion存在login_verifyCode，则说明login.jsp已经需要输入验证码，应该进行校验，如果错误返回信息到login.jsp回显

        4. 如果无异常，说明登录成功，

            * 为了安全应当先销毁原来可能存在的session，因为前一个用户可能并没有点击退出就关闭了页面，导致session还没死亡

            * 保存user到新的session中，删除session中的登录错误次数和验证码(防止下次一登录出错就要验证)，并跳转会原先要访问的页面

    3. Servcie层：UserService#login(User user)

        * 调用dao层findByUsername方法查询t_user

        * 如果t_user为空，则返回用户名错误的UserException异常信息

        * 如果t_user的state为false，则返回用户还未激活的UserException异常信息

        * 如果t_user的password和user的password不匹配，则返回密码错误的UserException异常信息

        * 登录错误则记录错误次数到session，如果登录错误次数超过3次，则login.jsp需要填写验证码

    4. dao层：UserDao#findByUsername(String username)

    5.filter层：过滤器

        * 用户如果需要购买物品，则需要登录，所以对这些操作要进行过滤，判断是否登录，如果为登录，跳转到登录页面，之后再跳转回来原先要访问的页面



  * 用户退出：

    * 可选的操作：如果session中的user保存了用户的一些影响了数据库的数据，比如积分之类的，应该在退出之前先把session的user更到数据库

    * 销毁session，并跳转到登录界面

    * 如果用户没有点击退出，而是直接关闭浏览页面，session还需要过了超时时间才会死亡，那么需要在session死亡时更新user到数据库

    

###1.2 分类模块

  * 页面

    * main.jsp：请求CategoryServlet#queryAll()

    * left.jsp：展示所有的书籍商品分类

  * Servlet层：CategoryServelt#queryAll()

    * 请求CategoryServcie#queryAll()方法、

    * 把所有的查询结果转发到leftl.jsp中显示

  * Service层：CategoryServelt#queryAll()

    * 请求CategoryDao#queryAll()方法

  * Dao层：CategoryDao#queryAll()

    * 查询数据库category表所有的数据分类条目



###1.3 图书模块

  * 相关类或者页面及其主要方法功能





###1.4 购物车模块：

  * 实现：使用Session来保存，虽然占内存容易丢失，但是想对于使用数据库来说更快速，想对于使用cookie来说更容易保存信息，但是Session中应该用怎么的形式来保存购物车，每个用户的购物车都应该区分，使用uid来作为key构建cartMap存放在session？？？？

  * 相关类或者页面及其主要方法功能

    * domian：双向关联CartItem <----> Cart

        * CartItem：购物车条目类，包含条目内容Book类book和条目数量count

        * Cart：购物车类

            * 属性：包含一个CartItemMap类，key为book条目的bid，value则为CartItem，还有总金额属性（有getTotal即可),注意金额运算的精确性，使用BigDecimal和double来运算

            * 方法：方法包括添加删除购物车条目等

    * web.servlet：CartServlet，继承于BaseServlet，主要有查看我的购物车，添加购物车条目，删除条目，清空购物车等方法

    * web.filter：CartServletFilter，客户机反问CartServelt前进行拦截，如果没有登录，则跳转登录，以及为等登录的用户进行Session中购物车类Cart的添加



  * 查看我的购物车

    * 页面： top.jsp发起请求，target设置在body的iframe中显示

    * Servlet层：CartServlet#lookMyCart()，把Session中的cart转发到/jsps/cart/list.jsp中在body框架中回显

  * 添加商品条目到购物车

    * 页面：

        * body框架中的/jsps/book/desc.jsp发起AJAX请求，把商品书籍的bid和数量count发送给CartServlet#addToCart

    * Servlet层：

        * CartServlet#addToCart()

            1.如果bid在cart中的cartItemMap已经存在，则只增加bid对应的cartItem中的count即可

            2.如果bid不在cart中存在，则使用bid利用boo.dao层的findByBid来生成Book类对象，添加到cart中

            3.向客户端写入OK，表示添加购物车成功

  * 删除购物车条目

     * 页面：body框架中/jps/cart/list.jsp中发起删除请求，发送商品bid

     * Servlet层：CartServlet#deleteCartItem()，根据bid来删除cart中的cartItem

  * 清空购物车

     * 页面：body框架中/jps/cart/list.jsp中发起清空请求

     * Servlet层：CartServlet#emptyCart()，删除cart中的cartItemMap



###1.5 订单模块





##2 后台模块








