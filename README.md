# mybatis-selectivePlugin
mybatis 插件查询指定列
1.针对mybatis 我们 逆向工程生成的代码  有个方法 selectByExample 
 

<select id="selectByExample" resultMap="BaseResultMap" parameterType="com.wuss.model.hotel.HotelCityInfoCriteria" >
    <!--
     WARNING -  该映射文件为自动生成, 请勿修改.
    -->
    select
    <if test="distinct" >
      distinct
    </if>
    <include refid="Base_Column_List" />
    from hotel_city_info
    <if test="_parameter != null" >
      <include refid="Example_Where_Clause" />
    </if>
    <if test="orderByClause != null" >
      order by ${orderByClause}
    </if>
    <if test="limit != 0 " >
       limit ${start} , ${limit}
    </if>
  </select>
 
  <sql id="Base_Column_List" >
    <!--
     WARNING -  该映射文件为自动生成, 请勿修改.
    -->
    id, city_name, city_code, full_pin_yin, short_pin_yin
  </sql>
 

那么查询的时候就等价于 select * ，

没法只查询指定列，加快查询，如果想要查询指定列 的时候手写sql 又比较麻烦，能否可以 灵活的只查询指定列？

因此本人写了一个 只查询指定列的工具类

@Intercepts(
        {
                @Signature(type = StatementHandler.class, method = "prepare", args = {Connection.class}),
        }
)
@Slf4j
public class SelectSpecificValue implements Interceptor {
 
//
    private String selectMethod;
    private boolean showLog;
 
    private Properties properties;
 
    /**
     * 需要查询数据库的指定列和excludeThreadLocal不能共存
     * @author: wuss@wjs.com
     * @date: 2021/5/11 4:19 PM
     * @return:
     */
    protected static final ThreadLocal<List<String>> threadLocal = new ThreadLocal<>();
    /**
     * 排除查询数据库的指定列 和threadLocal不能共存
     */
    protected static final ThreadLocal<List<String>> excludeThreadLocal = new ThreadLocal<>();
 
    public static void setSelectList(List<String> list) {
        if (CollectionUtils.isNotEmpty(list)) {
            threadLocal.set(list);
        }
    }
    public static void setExcludeSelectList(List<String> list){
        if (CollectionUtils.isNotEmpty(list)) {
            excludeThreadLocal.set(list);
        }
    }
 
    private boolean dealSelectListFlag(){
 
        List<String> selectList = threadLocal.get();
        List<String> excludeList = excludeThreadLocal.get();
        if (CollectionUtils.isNotEmpty(selectList) && CollectionUtils.isNotEmpty(excludeList)){
            throw new BaseException("查询指定列 和 排除指定列不能共存");
        }
 
        if (CollectionUtils.isNotEmpty(selectList)){
            return true;
        }
        if (CollectionUtils.isNotEmpty(excludeList)){
            return true;
        }
        return false;
    }
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
 
        RoutingStatementHandler target = (RoutingStatementHandler) invocation.getTarget();
 
        MetaObject metaObject = SystemMetaObject.forObject(target);
        BoundSql boundSql = (BoundSql) metaObject.getValue("delegate.boundSql");
        MappedStatement statement = (MappedStatement) metaObject.getValue("delegate.mappedStatement");
        String id = statement.getId();
        if (id.indexOf(selectMethod) < 0 || !dealSelectListFlag()) {
            return invocation.proceed();
        }
        StringBuilder sb = buildSql(boundSql);
 
        metaObject = SystemMetaObject.forObject(boundSql);
        metaObject.setValue("sql", sb.toString());
 
        if (showLog){
            log.info("查询指定列sql:{}", sb.toString());
        }
        sb.setLength(0);
        Object proceed = null;
        try {
            proceed = invocation.proceed();
        }finally {
            clearThreadLocal();
        }
        return proceed;
    }
 
    private void clearThreadLocal() {
        threadLocal.remove();
        excludeThreadLocal.remove();
    }
 
    private StringBuilder buildSql(BoundSql boundSql) {
        String sql = boundSql.getSql();
 
        int index = sql.indexOf("from ");
        if (index <0){
            log.info("sql 语句异常:{}",sql);
            throw new BaseException("sql 语句异常");
        }
 
        StringBuilder sb = new StringBuilder();
        if (CollectionUtils.isNotEmpty(threadLocal.get())){
            buildSelectThreadLocal(sql, index, sb);
            return sb;
        }
        buildExcludeSelectThreadLocal(sql,index,sb);
 
        return sb;
    }
 
    private void buildSelectThreadLocal(String sql, int index, StringBuilder sb) {
        sb.append("select ");
 
        for (String s : threadLocal.get()) {
            sb.append(s + ",");
        }
 
        sb.setLength(sb.length() - 1);
        sb.append(" ");
        sb.append(sql.substring(index));
    }
 
    private void buildExcludeSelectThreadLocal(String sql, int index, StringBuilder sb) {
        String preStr = sql.substring(0, index);
 
        for (String s : excludeThreadLocal.get()) {
            preStr = preStr.replaceFirst(",? *\t*"+s,"");
        }
 
        preStr = preStr.replaceFirst("select[ *\t*\n*]*,","select  ");
 
        sb.append(preStr);
        sb.append(" ");
        sb.append(sql.substring(index));
    }
 
 
    @Override
    public Object plugin(Object target) {
 
        return Plugin.wrap(target, this);
    }
 
    @Override
    public void setProperties(Properties properties) {
        this.properties = properties;
    }
 
    public String getSelectMethod() {
        return selectMethod;
    }
 
    public void setSelectMethod(String selectMethod) {
        this.selectMethod = selectMethod;
    }
 
    public boolean isShowLog() {
        return showLog;
    }
 
    public void setShowLog(boolean showLog) {
        this.showLog = showLog;
    }
}
 

 

在配置Bean的时候配置如下：

 

	<bean id="selectPlugin" class="com.wjs.tmc.util.SelectSpecificValue">
		<property name="selectMethod" value="selectByExample" />
	</bean>
 在配置 plugins
 

说明：

1.setSelectList 设置只查询指定字段

2.setExcludeSelectList 设置排除字段

且两者不能共存

为啥引入：ThreadLocal 为了线程安全 和 跨方法调用传参，查询出结果只会 remove 防止内存泄漏

使用：



 
————————————————
版权声明：本文为CSDN博主「走一步-再走一步」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/zy1404/article/details/116784242
