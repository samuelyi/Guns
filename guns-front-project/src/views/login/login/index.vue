<template>
  <div :class="['login-wrapper', { 'login-wrapper-responsive': styleResponsive }]" ref="loginPage">
    <div class="login-info" v-if="!isMinWidth">
      <h1 class="login-title">{{ themeInfo.gunsMgrName }}</h1>
      <h4 class="login-subtitle">{{ themeInfo.gunsSubTitle }}</h4>
    </div>
    <a-card class="login-card" :bordered="false">
      <div class="login-cover">
        <h1 class="login-title" v-if="isMinWidth">{{ themeInfo.gunsMgrName }}</h1>
        <h4 class="login-subtitle" v-if="isMinWidth">{{ themeInfo.gunsSubTitle }}</h4>
      </div>
      <div class="login-body">
        <h4 style="font-size: 32px; margin-bottom: 32px; font-weight: bold">{{ t('login.title') }}</h4>
        <a-radio-group v-model:value="tabActive" size="default" class="radio-group" v-show="false">
          <a-radio-button value="1">密码登录</a-radio-button>
        </a-radio-group>
        <!-- 登录 -->
        <a-form class="login-form">
          <!-- 密码登录 -->
          <a-form-item v-bind="validateInfos.account">
            <a-input
              allow-clear
              size="large"
              @keydown.enter="logoClick"
              v-model:value="form.account"
              :placeholder="t('login.username')"
              class="border-radius"
            >
              <template #prefix>
                <user-outlined />
              </template>
            </a-input>
          </a-form-item>
          <a-form-item v-bind="validateInfos.password">
            <a-input-password
              @keydown.enter="logoClick"
              size="large"
              v-model:value="form.password"
              :placeholder="t('login.password')"
              class="border-radius"
            >
              <template #prefix>
                <lock-outlined />
              </template>
            </a-input-password>
          </a-form-item>
          <a-form-item>
            <div class="register">
              <a-checkbox v-model:checked="form.rememberMe"> 7天免登陆 </a-checkbox>
            </div>
          </a-form-item>
          <a-form-item>
            <a-button
              block
              size="large"
              type="primary"
              :loading="loading"
              @click="logoClick"
              class="border-radius"
              style="height: 40px; margin-top: 20px"
            >
              {{ loading ? t('login.loading') : t('login.login') }}
            </a-button>
          </a-form-item>
          <a-form-item v-bind="validateInfos.verCode" v-if="captchaFlag">
            <div class="login-input-group">
              <a-input allow-clear size="large" v-model:value="form.verCode" :placeholder="t('login.code')" class="border-radius">
                <template #prefix>
                  <safety-certificate-outlined />
                </template>
              </a-input>
              <a-button class="login-captcha" @click="changeCaptcha">
                <img v-if="captcha" :src="captcha" alt="" />
              </a-button>
            </div>
          </a-form-item>
        </a-form>
      </div>
    </a-card>

    <!-- 图形验证 -->
    <Vertify
      v-model:visible="showVertify"
      v-if="showVertify"
      :validate="validate"
      :form="form"
      :ssoClientId="ssoClientId"
      :ssoCallback="ssoCallback"
      :goHome="goHome"
    />
  </div>
</template>

<script setup>
import { computed, onMounted, onUnmounted, reactive, ref, unref } from 'vue';
import { useI18n } from 'vue-i18n';
import { useRoute, useRouter } from 'vue-router';
import { Form, message } from 'ant-design-vue';
import { LockOutlined, SafetyCertificateOutlined, UserOutlined } from '@ant-design/icons-vue';
import { getToken, setToken } from '@/utils/token-util';
import { cleanPageTabs, goHomeRoute } from '@/utils/page-tab-util';
import { LoginApi } from '@/views/login/login/api/LoginApi';
import { CAPTCHA_FLAG, SSO_CLIENT_ID, SSO_FLAG, IS_NEED_RSA, DRAW_CAPTCHA_FLAG, CUSTOM_LOGIN_BACKGROUND_ENABLE } from '@/config/setting';
import { useSystemStore } from '@/store/modules/system';
import { SsoUtil } from '@/utils/common/sso-util';
import { RsaEncry } from '@/utils/common/util';
import { useThemeStore } from '@/store/modules/theme';
import { storeToRefs } from 'pinia';

const useForm = Form.useForm;
const themeStore = useThemeStore();
const { currentRoute } = useRouter();
const { query } = useRoute();
const { t } = useI18n();
const { styleResponsive } = storeToRefs(themeStore);

// 获取url传参的相关单点参数
const ssoClientId = query?.clientId;
const ssoCallback = query?.ssoCallback;

// guns添加的自定义配置
// 主题信息
let themeInfo = ref({
  gunsMgrLoginBackgroundImg: '',
  gunsMgrFooterText: '',
  gunsMgrBeiUrl: '',
  gunsMgrBeiNo: '',
  gunsMgrName: '',
  gunsSubTitle: ''
});
// 登录页面引用
const loginPage = ref(null);
// 验证码校验
const captchaFlag = ref(CAPTCHA_FLAG);
// 图形验证码校验
const drawCaptchaFlag = ref(DRAW_CAPTCHA_FLAG);
// 加载状态
const loading = ref(false);
// 表单数据
const form = reactive({
  account: '',
  password: '',
  verKey: '',
  verCode: '',
  rememberMe: false
});
// 租户注册数据
const tenantForm = reactive({});
// 验证码 base64 数据
const captcha = ref('');
// 验证码key标识
const verKey = ref('');

const showVertify = ref(false);

// 表单验证规则
let ruleLists = ref({
  account: [
    {
      required: true,
      message: t('login.username'),
      type: 'string',
      trigger: 'blur'
    }
  ],
  password: [
    {
      required: true,
      message: t('login.password'),
      type: 'string',
      trigger: 'blur'
    }
  ]
});
// 如果开启了验证码开关，则同时需要校验验证码是否填写
if (captchaFlag.value) {
  ruleLists.value.verCode = [
    {
      required: true,
      message: t('login.code'),
      type: 'string',
      trigger: 'blur'
    }
  ];
}

const { clearValidate, validate, validateInfos } = useForm(
  form,
  computed(() => ruleLists.value)
);

// 从store获取主题数据
let systemStore = useSystemStore();

// 当前登录方式
const tabActive = ref('1');

// 是否是最小宽度
const isMinWidth = ref(false);

/* 页面加载完成 */
onMounted(async () => {
  resizeChange();
  window.addEventListener('resize', resizeChange);
  let result = await systemStore.loadThemeInfo();
  themeInfo.value = result;
  // 动态设置登录页面的背景
  if (CUSTOM_LOGIN_BACKGROUND_ENABLE) {
    loginPage.value.style.setProperty('--customBackground', `url(${result.gunsMgrLoginBackgroundImg})`);
  }
});

// 屏幕大小改变
const resizeChange = () => {
  const width = window.innerWidth;
  if (width <= 680) {
    isMinWidth.value = true;
  } else {
    isMinWidth.value = false;
  }
};

onUnmounted(() => {
  window.removeEventListener('resize', resizeChange);
});

/* 跳转到首页 */
const goHome = () => {
  const { query } = unref(currentRoute);
  goHomeRoute(query.from);
};

// 登录按钮点击
const logoClick = () => {
  validate().then(() => {
    // 图形验证
    if (drawCaptchaFlag.value) {
      showVertify.value = true;
    } else {
      // 正常登录
      submit();
    }
  });
};

/* 提交 */
const submit = () => {
  loading.value = true;
  let formData = JSON.parse(JSON.stringify(form));

  // 是否需要加密
  if (IS_NEED_RSA) {
    // rsa加密密码
    formData.password = RsaEncry(formData.password);
  }

  // 如果开启了单点登录，则本系统就是单点登录服务端，调用获取loginCode接口
  if (SSO_FLAG) {
    LoginApi.getLoginCode(formData)
      .then(response => {
        SsoUtil.activateByLoginCode(ssoClientId ?? SSO_CLIENT_ID, ssoCallback ?? '', response?.data);
      })
      .finally(() => {
        loading.value = false;
      });
  }

  // 没开启单点登录，正常走登录接口逻辑
  else {
    LoginApi.login(formData)
      .then(response => {
        // 没开启单点登录
        message.success('登录成功');
        setToken(response?.data?.token, true);
        cleanPageTabs();
        goHome();
      })
      .finally(() => {
        loading.value = false;
      });
  }
};

/* 获取图形验证码 */
const changeCaptcha = () => {
  // 这里演示的验证码是后端返回base64格式的形式, 如果后端地址直接是图片请参考忘记密码页面
  LoginApi.getCaptcha()
    .then(response => {
      captcha.value = response.data.verImage;
      verKey.value = response.data.verKey;
      form.verKey = response.data.verKey;
      clearValidate();
    })
    .catch(e => {
      message.error(e.message);
    });
};

if (getToken()) {
  goHome();
} else {
  // changeCaptcha();
}
</script>

<style lang="less">
body {
  /*自定义背景图片*/
  --customBackground: url('@/assets/bg-login1.png');
}
</style>

<style lang="less" scoped>
.login-wrapper {
  min-height: 100vh;
  box-sizing: border-box;
  padding: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background-image: var(--customBackground);
  background-repeat: no-repeat;
  background-size: 100% 100%;
  position: relative;
}

.login-info {
  position: absolute;
  top: 8%;
  left: 6%;
}

.login-card {
  width: 80%;
  max-width: 100%;
  overflow: hidden;
  border-radius: 8px;
  background-color: transparent;
  :deep(.ant-card-body) {
    display: flex;
    padding: 0;
  }
}

.login-cover {
  flex: 1;
  padding: 36px 8px;
  box-sizing: border-box;
  background-repeat: no-repeat;
  background-position: bottom;
  background-size: contain;
  text-align: center;
}
.pick-center {
  background-position: center !important;
}

// 标题
.login-title {
  color: rgba(255, 255, 255, 1);
  font-size: 52px;
  margin: 0 0 6px 0;
  font-weight: normal;
  font-family: 'AliPuHui';
  letter-spacing: 1.2px;
}

.login-subtitle {
  color: rgba(255, 255, 255, 1);
  font-size: 32px;
  margin: 0;
  font-weight: normal;
  font-family: 'AliPuHui';
  letter-spacing: 4px;
}

.login-body {
  min-height: 450px;
  width: 400px;
  background-color: #fff;
  flex-shrink: 0;
  padding: 32px 48px 32px 48px;
  box-sizing: border-box;
  border-radius: 8px;
  margin: 10px;
  box-shadow: -6px 6px 10px 0px rgba(51, 65, 86, 0.15);
}

.login-form {
  height: calc(100% - 90px);
  display: flex;
  flex-direction: column;
  justify-content: center;
}

:deep(.ant-input) {
  font-size: 14px !important;
}

.radio-group {
  position: relative;
  margin-bottom: 8px;
  padding: 3px;
  border-radius: 5px;
  :deep(.ant-radio-button-input) {
    display: none;
  }
  :deep(.ant-radio-button-inner) {
    display: none;
  }
  .ant-radio-button-wrapper {
    font-size: 24px;
    border: 0;
    text-align: left;
    padding: 0;
    font-weight: 500;
    padding-bottom: 38px;
    color: rgba(0, 0, 0, 0.6);
    &:not(:last-child) {
      margin-right: 30px;
    }
  }
  .ant-radio-button-wrapper-checked {
    color: black;
    background: #fff;
    z-index: 3;
    border-radius: 3px;
    &:hover {
      background: #fff;
    }
  }
  :deep(.ant-radio-button-wrapper-checked:not(.ant-radio-button-wrapper-disabled)::before) {
    display: none;
  }
  :deep(.ant-radio-button-wrapper:not(:first-child)::before) {
    display: none;
  }

  :deep(.ant-radio-button-checked) {
    border-bottom: 4px solid #1890ff;
  }
}

// 小屏幕适应
@media screen and (max-width: 680px) {
  .login-wrapper-responsive {
    padding: 0;
    display: block;
    background: #fff;

    .login-card {
      width: 100%;
      background: none;
      box-shadow: none;
      border-radius: 0;

      :deep(.ant-card-body) {
        display: block;
        height: auto;
      }
    }

    .login-cover {
      padding: 24px 12px 100px 12px;
      background-size: auto 100px;
      background-repeat: no-repeat;
      background-size: 100% 100%;
      background-image: var(--customBackground);

      .login-title, .login-subtitle {
        text-shadow:
          0 0 10px #fff,
          /* 白色主发光 */ 0 0 20px #fff,
          /* 白色次发光 */ 0 0 30px #fff,
          /* 白色再次发光 */ 0 0 40px var(--primary-color),
          /* 粉色发光 */ 0 0 50px var(--primary-color),
          /* 粉色发光 */ 0 0 60px var(--primary-color),
          /* 粉色发光 */ 0 0 70px var(--primary-color); /* 粉色发光 */
      }
    }

    .login-body {
      width: 100%;
      box-shadow: none;

      .radio-group {
        margin-bottom: 32px;
      }
    }
  }
}

html.dark .login-wrapper {
  background: #000;
}
.register {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.login-input-group {
  display: flex;
  align-items: center;
}
.login-input-group .login-captcha {
  width: 130px;
  height: 40px;
  font-size: 14px;
  border-radius: 5px;
  padding: 0;
}
</style>
