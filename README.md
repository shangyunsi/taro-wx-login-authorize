# 小程序登陆鉴权

小程序打开鉴权登陆的方式目前大概有几种

* 1. 在request请求前进行鉴权
* 2. 在app.js中进行鉴权
* 3. 通过反向劫持生命周期进行鉴权

我们最后采用的是第三种, 第二种也有用过, 但是效果上感觉不好,第一种没有采用（由于不适用我们的场景,故没有采用)





### request 请求鉴权

适用场景

对于在用户未登陆的情况下,可以操作基本的查看浏览功能,而仅需对特殊操作进行鉴权（比如 新闻类小程序未登陆的情况下可以浏览基本信息流，当要评论等需要用户操作的时候需要鉴权）

实现方案

```js
  const token = Taro.getStorageSync('token');
  
  return Taro.request({
    url: `${config.server}${opts.url}`,
    data: options,
    method: opts.method || 'GET',
    header: {
      token,
    },
  }).then(({ data }) => {
    if (data && data.status === 'FAIL') {
      if (data.error_code === 'ERR_TOKEN_INVALID') {
        return Promise.reject(data);
      }
      return data;
    }
    return data;
  }).catch(data=>{
    if (data.error_code === 'ERR_TOKEN_INVALID') {
      redirectToLogin();
    }
    return Promise.reject(data);
  });
```

我们没有采用这种方案的原因是因为我们所有的请求都是需要用户来授权的,而page的生命周期是并发的,比如 onShow 和 onLoad, 如果我在一个page中的两个生命周期都发起了request请求,onload不会等到onshow结束再去执行,就会发出多个request,这时就会触发两次redirect事件,非常蛋疼～～～


### app.js 鉴权
这种方式是通过利用app.js的生命周期,在onShow的时候,去做鉴权操作,这样的弊端就是app的onshow和 page的生命周期是并行的,并不会阻塞页面的加载,并且需要等待page Loading结束之后才能跳转,这样就有可能出现页面请求已经发出,然后再进行跳转的事件 ～～～

实现方案

```js
    componentDidShow(){
        try {
            yield checkSessionPromise();
        } catch (err) {
            const res = yield loginPromise();
            const { token, uuid } = yield call(register,    res.code);
            Taro.setStorageSync('token', token);
            Taro.setStorageSync('uid', uuid);
        };

        try {
            const getSetting = promisify(Taro.getSetting);
            const getUserInfoPromise = promisify(Taro.getUserInfo);
            const setting = yield getSetting();
            if (setting.authSetting['scope.userInfo']) {
            const { userInfo } = yield getUserInfoPromise();
            const user = yield call(login, userInfo);

            if (user) {
                yield put({ type: 'save',
                payload: {
                    userInfo: user,
                    isLogin: true,
                },
                });
            } else {
                redirectToLogin();
            }
            } else {
                redirectToLogin();
            }
        } catch (err){
            redirectToLogin();
        }
    }
```


### 通过反向劫持生命周期进行鉴权

这里借鉴了[微信小程序授权登陆方案以及在Taro下利用Decorator修饰器实现](https://juejin.im/post/5b97a762e51d450e9649a8fd)

原文只对能对单生命周期进行鉴权,前文提到我们的场景可能会有多生命周期的情况,故对此做了一定的改造,具体看代码～～～
```js
/* eslint-disable camelcase */
const LIFE_CYCLE_MAP = ['willMount', 'didMount', 'didShow'];

/**
 *
 * 登录鉴权
 *
 * @param {string} [lifecycle] 需要等待的鉴权完再执行的生命周期 willMount didMount didShow
 * @returns 包装后的Component
 *
 */
function withLogin(lifecycle = ['willMount']) {
  if (!(lifecycle instanceof Array)) {
    return;
  }
  const lifecycleArr = lifecycle.filter(cycle=>LIFE_CYCLE_MAP.indexOf(cycle) >= 0);
  // 异常规避提醒
  if (lifecycleArr.length === 0) {
    console.warn(
      '传入的生命周期不存在, 鉴权判断异常 ===========> $_{lifecycle}',
    );
    return Component => Component;
  }

  return function withLoginComponent(Component) {
    // 这里还可以通过redux来获取本地用户信息，在用户一次登录之后，其他需要鉴权的页面可以用判断跳过流程
    // @connect(({ app }) => ({
    //   app,
    // }))
    return class WithLogin extends Component {
      constructor(props) {
        super(props);
      }

      async componentWillMount() {
        if (this.$_login) {
          if (super.componentWillMount) {
            this.$_cycle.push(super.componentWillMount);
          }
          return;
        }
        const { app } = this.props;
        const { isLogin } = app;
        if (super.componentWillMount) {
          if (lifecycleArr.some(cycle=>cycle === LIFE_CYCLE_MAP[0]) && !isLogin) {
            const res = await this.$_autoLogin();
            this.$_login = false;
            if (res) {
              this.$_cycle.forEach(cycle=>{
                cycle.call(this);
              });
              super.componentWillMount();
            }
            return;
          }

          super.componentWillMount();
        }
      }

      async componentDidMount() {
        if (this.$_login) {
          if (super.componentDidMount) {
            this.$_cycle.push(super.componentDidMount);
          }
          return;
        }
        const { app } = this.props;
        const { isLogin } = app;
        if (super.componentDidMount) {
          if (lifecycleArr.some(cycle=>cycle === LIFE_CYCLE_MAP[1]) && !isLogin) {
            const res = await this.$_autoLogin();
            this.$_login = false;
            if (res) {
              this.$_cycle.forEach(cycle=>{
                cycle.call(this);
              });
              super.componentDidMount();
            }
            return;
          }

          super.componentDidMount();
        }
      }

      async componentDidShow() {
        if (this.$_login) {
          if (super.componentDidShow) {
            this.$_cycle.push(super.componentDidShow);
          }
          return;
        }
        const { app } = this.props;
        const { isLogin } = app;
        if (super.componentDidShow) {
          if (lifecycleArr.some(cycle=>cycle === LIFE_CYCLE_MAP[2]) && !isLogin) {
            const res = await this.$_autoLogin();
            this.$_login = false;
            if (res) {
              this.$_cycle.forEach(cycle=>{
                cycle.call(this);
              });
              super.componentDidShow();
            }
            return;
          }

          super.componentDidShow();
        }
      }

      $_autoLogin = async () => {
        this.$_login = true;
        const { dispatch } = this.props;
        return await dispatch({ type: 'app/checkLogin' });
      };


      $_login = false;

      $_cycle = [];
    };
  };
}

export default withLogin;

```



