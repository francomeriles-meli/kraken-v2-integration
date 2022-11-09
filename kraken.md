# Kraken v2 Integration

> Integrate kraken v2 navbar in legacy pages and nordic pages

## Requirements 

 - Kraken v2.1+ and Odin v2.4+
 - Have the functional application under the adminml domain
 - Have 100% ldap users configured with full attributes (xd,wharehouse,sc,site,region)
 - Nordic v7+

## Install

Install the necessary packages `npm i @kraken/core odin-security --save`

## Integration

In the file **config/default.js** add the configurations for each scope you need

```js
  auth_api_scope: 'sandbox',
  menu_api_scope: 'sandbox',
  attributes_api_scope: 'sandbox',
  productName: 'mercadoenvios', // Menu name 
  applicationName: 'fos-divergences-frontend', // Your app name 
```

**config/prod.js**

```js
  auth_api_scope: 'prod',
  menu_api_scope: 'prod',
  attributes_api_scope: 'prod',
```

In the file **app/server/index.js**

Before any middleware add the authentication middleware from odin

```js
import { authentication } from 'odin-security';

router.use(authentication());
```

After that, add the kraken layout

```js
import { appNavMiddleware } from '@kraken/app-nav';
import { kraken } from '@kraken/core';
import i18nMiddleware from 'nordic/i18n/middleware';
import config from 'nordic/config';

router.use([
  kraken.release({
    authorization: {
      newAttributesApi: true,
    },
    layout: {
      baseApiPath: '', // Your API Base Path: example '/api/fos/divergences'
      enableNotifications: true,
    },
  }),
  i18nMiddleware(config.i18n),
  appNavMiddleware({
    headerAttribute: ['site'],
    attributes: ['site', 'region', 'xd', 'sc', 'warehouse'],
  }),
]);
```

In your GetServerSideProp (Nordic Pages) or Controller (Legacy Pages) from the View add the necesary properties

```js
export const getServerSideProps = async req => {
  const { i18n, krakenAppNavConfig } = req;

  const selectorModule = [
    // This module is for display site selector in the header modal
    {
      name: 'attributessite', 
      host: 'https://kraken-frm-beta.adminml.com',
      attributeKey: 'site',
      multipleSelection: false,
    },
    // This module is for display region selector in the header modal
    {
      name: 'attributesregion',
      host: 'https://kraken-frm-beta.adminml.com',
      attributeKey: 'region',
      parentAttributeKey: 'site',
      multipleSelection: false,
    },
    // This module is for display wharehouses selector in the header modal
    {
      name: 'attributeswarehouse',
      host: 'https://kraken-frm-beta.adminml.com',
      attributeKey: 'warehouse',
      parentAttributeKey: 'site',
      multipleSelection: false,
    },
    // This module is for display service center selector in the header modal
    {
      name: 'attributesservicecenter',
      host: 'https://kraken-frm-beta.adminml.com',
      attributeKey: 'sc',
      parentAttributeKey: 'region',
      multipleSelection: false,
    },
    // This module is for display cross docking selector in the header modal
    {
      name: 'attributesfacility',
      host: 'https://kraken-frm-beta.adminml.com',
      attributeKey: ['xd', 'warehouse'],
      parentAttributeKey: 'site',
    },
  ];

  return {
    props: {
      appNavConfig: krakenAppNavConfig,
      appNavModules: selectorModule,
      userAttributes: req.user.attributes,
    }
  };
};
```

In the View add the Provider with the configurations

```js
export const Home = props => {
  const { userAttributes, appNavConfig, appNavModules } = props;

  return (
    <Context.AppNavProvider
      config={appNavConfig}
      modules={appNavModules}
      userAttributes={userAttributes}
    >
     <YourView />
    </Context.AppNavProvider>
  );
};
```
Then we need to detect each change when a user changes the logistics center or the site

Create a new component in your componentes folder for example **api/components/HeaderAction/index.js**

```js
import React from 'react';
import PropTypes from 'prop-types';
import { Context } from '@kraken/app-nav';

const { injectAppNavContext } = Context;

const HeaderActions = ({ appNavContext, children }) => {
  React.useEffect(() => {
    console.log('Context nav changed');
    console.log(appNavContext.appNavData);
    // Here you can handle all changes
    // for example set/update logistic-center and site-id cookie

  }, [appNavContext.appNavData]);

  return <>{children}</>;
};

HeaderActions.propTypes = {
  children: PropTypes.node,
  appNavContext: PropTypes.shape({ appNavData: PropTypes.shape({}) })
    .isRequired,
};

export default injectAppNavContext(HeaderActions);

```

And add this componente right after the AppNavProvier

```js
export const Home = props => {
  const { userAttributes, appNavConfig, appNavModules } = props;

  return (
    <Context.AppNavProvider
      config={appNavConfig}
      modules={appNavModules}
      userAttributes={userAttributes}
    >
      <HeaderAction>
        <YourView />
      </HeaderAction>
    </Context.AppNavProvider>
  );
};
```

Finally we need to add some middlewares in the internal API

In the file **api/index.js**

```js
const { preferencesRouter } = require('@kraken/user-setup');
const { krakenUserSetupMiddleware } = require('@kraken/user-setup');
const { authentication } = require('odin-security');

router.use(authentication());
router.use(krakenUserSetupMiddleware());

router.use('/', preferencesRouter);
```

