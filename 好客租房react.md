@[TOC]
# React入门
## mock dva
react/config/config.js
``` javascript
export default {
    plugins:[
        ['umi-plugin-react',{
            dva: true,   //开启dva进行数据分层管理
        }]
    ]
};
```
react/mock/MockListData.js

``` javascript
export default {
    'get /ds/list': function (req, res) { //模拟请求返回数据
        res.json({
            data: [1, 2, 3, 4, 5],
            maxNum: 5
        });
    }
}
```
react/src/util/request.js

``` qml
// import fetch from 'dva/fetch';

function checkStatus(response) {
    if (response.status >= 200 && response.status < 300) {
        return response;
    }

    const error = new Error(response.statusText);
    error.response = response;
    throw error;
}

/**
 * Requests a URL, returning a promise.
 *
 * @param  {string} url       The URL we want to request
 * @param  {object} [options] The options we want to pass to "fetch"
 * @return {object}           An object containing either "data" or "err"
 */
export default async function request(url, options) {
    const response = await fetch(url, options);
    checkStatus(response);
    return await response.json();
}
```
react/src/models/ListData.js

``` pf
import request from '../util/request';

export default {
    namespace: 'list',
    state: {
        data: [],
        maxNum: 1
    },
    reducers : { // 定义的一些函数
        addNewData : function (state, result) { // state：指的是更新之前的状态数据, result: 请求到的数据

            if(result.data){ //如果state中存在data数据，直接返回，在做初始化的操作
                return result.data;
            }

            let maxNum = state.maxNum + 1;
            let newArr = [...state.data, maxNum];


            return {
                data : newArr,
                maxNum : maxNum
            }
            //通过return 返回更新后的数据
        }
    },
    effects: { //新增effects配置，用于异步加载数据
        *initData(params, sagaEffects) { //定义异步方法
            const {call, put} = sagaEffects; //获取到call、put方法
            const url = "/ds/list"; // 定义请求的url
            let data = yield call(request, url); //执行请求
            yield put({ // 调用reducers中的方法
                type : "addNewData", //指定方法名
                data : data //传递ajax回来的数据
            });
        }
    }
}
```
react/src/pages/List.js

``` javascript
import React from 'react';
import { connect } from 'dva';

const namespace = "list";

// 说明：第一个回调函数，作用：将page层和model层进行链接，返回modle中的数据,并且将返回的数据，绑定到this.props
// 接收第二个函数，这个函数的作用：将定义的函数绑定到this.props中，调用model层中定义的函数
@connect((state) => {
    return {
        dataList : state[namespace].data,
        maxNum : state[namespace].maxNum
    }
}, (dispatch) => { // dispatch的作用：可以调用model层定义的函数
    return { // 将返回的函数，绑定到this.props中
        add : function () {
            dispatch({ //通过dispatch调用modle中定义的函数,通过type属性，指定函数命名，格式：namespace/函数名
                type : namespace + "/addNewData"
            });
        },
        init : () => {
            dispatch({ //通过dispatch调用modle中定义的函数,通过type属性，指定函数命名，格式：namespace/函数名
                type : namespace + "/initData"
            });
        }
    }
})
class  List extends React.Component{

    componentDidMount(){
        //初始化的操作
        this.props.init();
    }

    render(){
        return (
            <div>
                <ul>
                    {
                        this.props.dataList.map((value,index)=>{
                            return <li key={index}>{value}</li>
                        })
                    }
                </ul>
                <button onClick={() => {
                    this.props.add();
                }}>点我</button>
            </div>
        );
    }

}

export default List;
```
umi dev
# Ant Design 入门
react/config/config.js
``` less
export default {
    plugins:[
        ['umi-plugin-react',{
            dva: true,   //开启dva进行数据分层管理
            antd: true // 开启Ant Design功能
        }]
    ],
    routes: [{
        path: '/',
        component: '../layouts', //配置布局路由
        routes: [
            {
                path: '/',
                component: './index'
            },
            {
                path: '/myTabs',
                component: './myTabs'
            },
            {
                path: '/user',
                routes: [
                    {
                        path: '/user/list',
                        component: './user/UserList'
                    },
                    {
                        path: '/user/add',
                        component: './user/UserAdd'
                    }
                ]
            }
        ]
    }]
};
```
react/mock/MockListData.js
 

``` xquery
   'get /ds/user/list': function (req, res) {
        res.json([{
            key: '1',
            name: '张三1',
            age: 32,
            address: '上海市',
            tags: ['程序员', '帅气'],
        }, {
            key: '2',
            name: '李四2',
            age: 42,
            address: '北京市',
            tags: ['屌丝'],
        }, {
            key: '3',
            name: '王五3',
            age: 32,
            address: '杭州市',
            tags: ['高富帅', '富二代'],
        }]);
```
react/src/models/UserListData.js

``` typescript
import request from "../util/request";

export default {
    namespace: 'userList',
    state: {
        list: []
    },
    effects: {
        *initData(params, sagaEffects) {
            const {call, put} = sagaEffects;
            const url = "/ds/user/list";
            let data = yield call(request, url);
            yield put({
                type : "queryList",
                data : data
            });
        }
    },
    reducers: {
        queryList(state, result) {
            let data = [...result.data];
            return { //更新状态值
                list: data
            }
        }
    }
}
```

react/src/layouts/index.js

``` vbscript-html
import React from 'react';
import { Layout, Menu, Icon } from 'antd';
import Link from 'umi/link';
const { Header, Footer, Sider, Content } = Layout;
const SubMenu = Menu.SubMenu;
// layouts/index.js文件将被作为全 局的布局文件。
class BasicLayout extends React.Component{
    constructor(props){
        super(props);
        this.state = {
            collapsed: true,
        }
    }
    render(){
        return (
            <Layout>
                <Sider width={256} style={{minHeight: '100vh', color: 'white'}}>
                    <div style={{ height: '32px', background: 'rgba(255,255,255,.2)', margin: '16px'}}/>
                    <Menu
                        defaultSelectedKeys={['1']}
                        defaultOpenKeys={['sub1']}
                        mode="inline"
                        theme="dark"
                        inlineCollapsed={this.state.collapsed}
                    >
                        <SubMenu key="sub1" title={<span><Icon type="user"/><span>用户管理</span></span>}>
                            <Menu.Item key="1"><Link to="/user/add">新增用户</Link></Menu.Item>
                            <Menu.Item key="2"><Link to="/user/list">新增列表</Link></Menu.Item>
                        </SubMenu>
                    </Menu>
                </Sider>
                <Layout>
                    <Header style={{ background: '#fff', textAlign: 'center', padding: 0 }}>Header</Header>
                    <Content style={{ margin: '24px 16px 0' }}>
                        <div style={{ padding: 24, background: '#fff', minHeight: 360 }}>
                            { this.props.children }
                        </div>
                    </Content>
                    <Footer style={{ textAlign: 'center' }}>后台系统</Footer>
                </Layout>
            </Layout>
        )
    }
}
export default BasicLayout;
```
react/src/pages/MyTabs.js

``` 
import React from 'react';
import { Tabs } from 'antd'; // 第一步，导入需要使用的组件

const TabPane = Tabs.TabPane;

function callback(key) {
    console.log(key);
}

class MyTabs extends React.Component{

    render(){
        return (
            <Tabs defaultActiveKey="1" onChange={callback}>
                <TabPane tab="Tab 1" key="1">hello antd wo de 第一个 tabs</TabPane>
                <TabPane tab="Tab 2" key="2">Content of Tab Pane 2</TabPane>
                <TabPane tab="Tab 3" key="3">Content of Tab Pane 3</TabPane>
            </Tabs>
        )
    }

}

export default MyTabs;
```
react/src/pages/user/UserList.js

``` scala
import React from 'react';
import { connect } from 'dva';

import {Table, Divider, Tag, Pagination } from 'antd';

const {Column} = Table;

const namespace = 'userList';

@connect((state)=>{
    return {
        data : state[namespace].list
    }
}, (dispatch) => {
    return {
        initData : () => {
            dispatch({
                type: namespace + "/initData"
            });
        }
    }
})
class UserList extends React.Component {

    componentDidMount(){
        this.props.initData();
    }

    render() {
        return (
            <div>
                <Table dataSource={this.props.data} pagination={{position:"bottom",total:500,pageSize:10, defaultCurrent:3}}>
                    <Column
                        title="姓名"
                        dataIndex="name"
                        key="name"
                    />
                    <Column
                        title="年龄"
                        dataIndex="age"
                        key="age"
                    />
                    <Column
                        title="地址"
                        dataIndex="address"
                        key="address"
                    />
                    <Column
                        title="标签"
                        dataIndex="tags"
                        key="tags"
                        render={tags => (
                            <span>
                                {tags.map(tag => <Tag color="blue" key={tag}>{tag}</Tag>)}
                            </span>
                        )}
                    />
                    <Column
                        title="操作"
                        key="action"
                        render={(text, record) => (
                            <span>
                                <a href="javascript:;">编辑</a>
                                <Divider type="vertical"/>
                                <a href="javascript:;">删除</a>
                            </span>
                        )}
                    />
                </Table>
            </div>
        );
    }

}

export default UserList;
```
react/src/pages/user/UserAdd.js

``` scala
import React from 'react'
class UserAdd extends React.Component{
    render(){
        return (
            <div>新增用户</div>
        );
    }

}
export default UserAdd;
```
react/src/pages/index.js

``` 
import React from 'react'
class Index extends React.Component {
    render(){
        return <div>首页</div>
    }
}
export default Index;
// http://localhost:8000/
```

详情见：
https://github.com/OneJane/blog
