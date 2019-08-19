@[TOC]
# mock dva
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

详情见：
https://github.com/OneJane/blog
