---
title: 微信小程序（六）——ThinkPHP接口篇
date: 2018-01-20
categories: "Android"
tags: "小程序"
---
# 集成ThinkPHP5
我们在{% post_link 微信小程序（二）——开发工具篇 %}下载完整版的ThinkPHP5，集成到我们的XAMPP里面去，按照上一节的内容，在D:\xampp\htdocs文件夹下新建thinkphp5文件夹，将下载好的THinkPHP5解压到该文件夹下，完成后如下：
![](http://oxr4g4c3v.bkt.clouddn.com/thinkphplujing.jpg)
 <!--more--> 
然后开启我们的XAMPP里面的Apach服务器：
![](http://oxr4g4c3v.bkt.clouddn.com/startapach.jpg)
在我们的浏览器中输入http://localhost/thinkphp5/public/ ，如果出现下面的界面，说明thinkphp已经集成部署成功
![](http://oxr4g4c3v.bkt.clouddn.com/thinkphppublic.jpg)

# 配置数据库
database.php在..\application中，注意表前缀，具体设置：
```
// 数据库类型
   'type'            => 'mysql',
   // 服务器地址
   'hostname'        => '127.0.0.1',
   // 数据库名
   'database'        => 'test',
   // 用户名
   'username'        => 'root',
   // 密码
   'password'        => '123456',
   // 端口
   'hostport'        => '',
   // 连接dsn
   'dsn'             => '',
   // 数据库连接参数
   'params'          => [],
   // 数据库编码默认采用utf8
   'charset'         => 'utf8',
   // 数据库表前缀
   'prefix'          => 'test_',
   // 数据库调试模式
   'debug'           => true,
   // 数据库部署方式:0 集中式(单一服务器),1 分布式(主从服务器)
   'deploy'          => 0,
   // 数据库读写是否分离 主从式有效
   'rw_separate'     => false,
   // 读写分离后 主服务器数量
   'master_num'      => 1,
   ```
# 配置文件
config.php也在..\application中，打开log：
```
// 应用调试模式
  'app_debug'              => true,
```
写一个测试接口
在application下新建index文件，在index文件夹新建controller文件下，在controller文件下新建一个公共父类Common.php，里面存放初始化方法和参数，再新建一个子类User.php，继承自Common，其用来处理用户、活动等逻辑：
![](http://oxr4g4c3v.bkt.clouddn.com/jiagou.jpg)

我们在User.php定义一个测试方法：
```php
public function index()
   {
       echo "这是测试返回";
   }
```
然后在浏览器中输入：http://localhost/thinkphp5/public/index/user/index ，查看输出结果，显示如下为正常：
![](http://oxr4g4c3v.bkt.clouddn.com/ceshijiekou.jpg)
这里说明一下http://localhost/thinkphp5/public/index/user/index ，最后的index是对应的方法名，如果你定义的方法名是test(),那么访问url就是http://localhost/thinkphp5/public/index/user/test ，最后跟的是test。

# 接口讲解
首先看Common.php：
任何接口在处理主要逻辑之前都要检验参数是否合法。所以将其统一写到Common里面，这里的$rules需要注意，”require”代表必需参数，”require|number”代表参数必需为数字，thinkphp官网有具体详解
```php
class Common extends Controller{
	protected $request;//处理参数
	protected $params;
    protected $rules = array(//验证参数的合法性
	 	'User' => array(
            'login'           => array(//对应的方法名，利用openid可完成登录
                'openid' => 'require',
            ),
            'leave'        => array(//申请加班
                'activity_openid' => 'require',//申请人id
                'activity_type'  => 'require',//申请类型，1还是2请假还是出差
                'activity_st'  => 'require',//开始时间
                'activity_et'  => 'require',//结束时间
                'activity_reson'  => 'require',
                'activity_fristcheck_openid' => 'require',
                'activity_time' => 'require|number',//活动时长
            ),
            'getactivitylist'   => array(//获取向我申请的活动，根据个人openid获取
                'openid' => 'require',
            ),
             'handleactivity'   => array(//处理一个活动
                'openid' => 'require',
                'check_one_or_two' => 'require',
                'activity_id' => 'require',
                'handle_id' => 'require',//handid
                'check_result' => 'require',//处理结果1是同意，2是拒绝
            ),
             'getmyactivitylist'   => array(//获取我的申请记录，根据个人openid获取
                'activity_openid' => 'require',
            ),
             'search'   => array(//搜索
                 'key' => 'require',
            ),
            'getsinglemyactivity'   => array(//获取单个活动
                 'activity_id' => 'require',
            ),
            'regiter'   => array(//注册
                'name' => 'require',
                'num' => 'require',
                'openid' => 'require',
            ),
        ),
	 );
	/**
	 *验证参数
	 */
	protected function _initializa(){
		parent::_initialize();
		$this -> request = Request::instance();
		$this->check_params($this->request->param(true));
	}
	/**
     * 验证参数 参数过滤
     * @param  [array] $arr [除time和token外的所有参数]
     * @return [return]      [合格的参数数组]
     */
    public function check_params($arr) {
        /*********** 获取参数的验证规则  ***********/
        $rule = $this->rules[$this->request->controller()][$this->request->action()];
        /*********** 验证参数并返回错误  ***********/
        $this->validater = new Validate($rule);
        if (!$this->validater->check($arr)) {
            $this->return_msg(400, $this->validater->getError());
        }
        /*********** 如果正常,通过验证  ***********/
         $this->params =  $arr;
    }
	
	/**
	 *检验openid是否为空
	 */
	public function check_Openid($arr){
		if(!isset($arr['openid'])){
			$this -> return_msg(400,'openid不正确');
		}
		 $this->paramsOpenid = $arr['openid'];
	}
	
	/**
	 *返回json数据的封装,例如：
	 {
	    "code":"200",
	    "msg":"访问成功"
	    "data":{
		
		       }
	 }
	 */
	public function return_msg($code,$msg = '',$data = []){
		$return_data['code'] = $code;
		$return_data['msg'] = $msg;
		$return_data['data'] = $data;
		echo json_encode($return_data);die;
	}
	/**
	 *检验参数是否存在
	 */
	public function check_exist($value) {
        $openid_res = db('user')->where('user_openid', $value)->find();
        return $openid_res;
    }
}
```
User.php用来处理主要的逻辑：
```php
class User extends Common{
    public function index()
    {
        echo "这是测试返回";
    }
	//启动先进行openid的判断，
	//如果openid不存在数据库，说明用户未注册
	//如果存在的话可正常进入程序
	public function login(){
		parent::_initializa();
		$data = $this->params;
		//检查opneid是否存在，不存在就去注册，存在则登录成功，返回姓名职称，工号
		if ( $this->check_exist($data["openid"])) {
				$db_res = Db::view('user','user_openid,user_name,user_num')
   						->view('position','position_name','position_id=user.user_pid')
   						->view('department','department_name','department_id=user.user_departmentid')
    					->where('user_openid', $data["openid"])
    					->find();
                $this->return_msg(200, '登录成功',$db_res);
            }else{
            	$this->return_msg(400, '未注册');
            }
	}
	//请假
	public function leave(){
		parent::_initializa();
		$data = $this->params;
        $res = db('activity') ->insertGetId($data);//插入数据返回id
        if ($res) {
            if($data["activity_secondcheck_openid"]==""||!isset($data["activity_secondcheck_openid"])){//插入单条数据，即活动没有第二审核人
                $arr = array(
                    'handle_check_uid' => $data["activity_fristcheck_openid"],
                    'handle_one_or_two' => 1,
                    'handle_check_activityid' => $res,
                );
                $re = db('handle') ->insertGetId($arr);//插入数据返回id
                if($re){
                    $this->return_msg(200, '新增成功!',$res);
                } else {
                    $this->return_msg(400, '新增失败!');
                }
            }else{//插入多条数据,有两个审核人存在的情况
                $dataall = [
                    ['handle_check_uid' => $data["activity_fristcheck_openid"], 'handle_check_activityid' => $res, 'handle_one_or_two' => 1],
                    ['handle_check_uid' => $data["activity_secondcheck_openid"], 'handle_check_activityid' => $res,'handle_one_or_two' => 2]
                ];
                $reAll = Db::name('handle')->insertAll($dataall);
                if($reAll){
                    $this->return_msg(200, '新增成功!',$reAll);
                } else {
                    $this->return_msg(400, '新增失败!');
                }
            }
        }
	}
	//获取向我申请的活动
	public function getactivitylist(){
		parent::_initializa();
		$data = $this->params;
		$db_res = Db::view('handle','handle_id,handle_check_uid,handle_check_activityid,handle_one_or_two')//多表查询
   						->view('activity','activity_id,activity_type,activity_st,activity_et,activity_openid,activity_secondcheck_openid','activity_id=handle.handle_check_activityid')
                        ->view('user','user_name,user_openid','activity_openid=user.user_openid')
    					->where('handle_check_uid', $data["openid"])
    					->select();
    	 $this->return_msg(200, '查询成功',$db_res);
	}
	//处理向我申请的活动
	public function handleactivity(){
		parent::_initializa();
		$data = $this->params;
        $db_r = db('activity')
            ->field('activity_secondcheck_openid,activity_fristcheck_openid')
            ->where('activity_id', $data["activity_id"])
            ->find();
        if($data['openid']!=$db_r['activity_secondcheck_openid']&&$data['openid']!=$db_r['activity_fristcheck_openid']){
            $this->return_msg(400, '你无权审核该申请');
        }
        if($data['check_result']!=2){
            if($data['check_one_or_two']==1){//第一个审核人的审核结果
                $res = db('activity')->where('activity_id',$data['activity_id'])->setField('activity_fristcheck_result',$data['check_result']);//1是拒绝2是同意
            }else{//第二个审核人的结果
                $res = db('activity')->where('activity_id',$data['activity_id'])->setField('activity_secondcheck_result',$data['check_result']);//1是拒绝2是同意
            }
            if($res){
                db('handle')->delete($data['handle_id']);//需要删除handle里面的数据
            }
            //判断整个活动是否已经完成
            $db_handleAll = db('handle')
                ->where('handle_check_activityid', $data["activity_id"])
                ->count();
            if($db_handleAll==0){//已经完成
                $ress = db('activity')->where('activity_id',$data['activity_id'])->setField('activity_allresult',2);//
                $this->return_msg(200, "此申请已经通过",$db_handleAll);
            }
            $this->return_msg(200, "处理完成",$db_handleAll);
        }else{//如果有一个审核人不同意，则整个申请都为不通过
//            $re = db('handle')->delete($data['activity_id']);//这里删除要根据activity_id删除一个或者多个记录
            db('handle')->where('handle_check_activityid',$data['activity_id'])->delete();
            $ress = db('activity')->where('activity_id',$data['activity_id'])->setField('activity_allresult',3);//
            $this->return_msg(200, "此条审核不通过");
        }
	}
	//获取我发出的申请
	public function getmyactivitylist(){
		parent::_initializa();
		$data = $this->params;
		$db_res = db('activity')
			->field('activity_id,activity_openid,activity_type,activity_st,activity_et,activity_secondcheck_openid')
			->where('activity_openid', $data["activity_openid"])
            ->order('activity_id desc')
			->select();    
		 $this->return_msg(200, '查询成功',$db_res);    		
	}
	//获取单个我发出的申请
    public function getsinglemyactivity(){
        parent::_initializa();
        $data = $this->params;
        $db_res = Db::view('activity','activity_id,activity_openid,activity_type,activity_st,activity_et,activity_address,activity_reson,activity_fristcheck_openid,activity_secondcheck_openid,activity_fristcheck_result,activity_secondcheck_result,activity_time,activity_allresult')//多表查询
        ->view('user','user_name','activity_fristcheck_openid=user.user_openid')
            ->where('activity_id', $data["activity_id"])
            ->find();
        $db_res2 = db('user')
            ->field('user_name')
            ->where('user_openid', $data["activity_secondcheck_openid"])->find();
        $db_res3 = db('user')
            ->field('user_name')
            ->where('user_openid', $db_res["activity_openid"])->find();
        $db_res['secondname'] = $db_res2;
        $db_res['name'] = $db_res3;
        $this->return_msg(200, '查询成功',$db_res);
    }
	
	/**
	 *模糊搜索审核人，可根据姓名或是工号
	 */
	public function search(){
        parent::_initializa();
        $data = $this->params;
        $where["user_name"] = ['like',"%".$data["key"]."%"];
        $map['user_num'] = ['like',"%".$data["key"]."%"];
        $db_res = Db::view('user','user_name,user_openid,user_pid,user_num')//多表查询
        ->view('position','position_id,position_name','user_pid=position.position_id')
            ->where($where)
            ->whereOr($map)
            ->select();
        $this->return_msg(200, '查询成功',$db_res);
    }
    /**
     * 注册
     */
    public function regiter(){
        parent::_initializa();
        $data = $this->params;
        $db_res = Db::view('user','user_id,user_name,user_num')
            ->where('user_num', $data["num"])
            ->find();
        if($db_res["user_name"]!=$data["name"]){
            $this->return_msg(400, '工号姓名不一致，注册失败');
        }else{
            $res = db('user')->where('user_id',$db_res["user_id"])->setField('user_openid',$data["openid"]);
            if($res){
                $db_res = Db::view('user','user_openid,user_name,user_num')
                    ->view('position','position_name','position_id=user.user_pid')
                    ->view('department','department_name','department_id=user.user_departmentid')
                    ->where('user_openid', $data["openid"])
                    ->find();
                $this->return_msg(200, '注册成功',$db_res);
            } else {
                $this->return_msg(400, '已注册',$res);
            }
        }
    }
}
```

# 资源下载
参看 {% post_link 微信小程序（九）——资源下载 %}
