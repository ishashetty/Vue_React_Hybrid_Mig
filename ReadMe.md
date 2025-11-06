# Incremental Migration from Vue to React

A practical guide for gradually migrating a large Vue application to React while keeping the existing app running.

## Introduction

In our recent work, we were tasked with exploring the migration of our application from Vue to React. The existing Vue feature was large and complex, so a complete migration wasn't feasible, especially since we couldn't dedicate full-time resources to the process.

To address this, we opted for a **gradual migration strategy**. This approach allowed us to keep the Vue application running in its current state while incrementally migrating one component at a time to React. Each React component would then be embedded and rendered within the Vue app.
Once all the components are migrated to React, we will finally replace the Vue version with its React equivalent.
While the idea seemed straightforward in theory, the implementation posed several challenges. One such challenge was integrating the React app—built using Create React App—into the Vue environment.

Additionally, we had to carefully manage state across both frameworks. The React application used Redux, while the Vue app relied on Vuex for state management. Coordinating these two state management systems required additional planning and consideration.

### Frameworks Used

- **React** - Target framework for new components
- **Redux** - State management for React
- **Vue** - Existing application framework
- **Vuex** - State management for Vue

---

## Getting Started

### Component Architecture

In both our React and Vue project structures, a consistent component-based design pattern was followed with a major focus on building reusable components to ensure consistency, promote code reusability, and simplify maintenance.

The first step in the migration process was started by identifying such reusable components. Thanks to this modular architecture, laying the foundation for the migration felt achievable within the given timeframe.

### Migration Steps Overview

1. **React Build Setup** - Configure webpack for embedding
2. **React Component Handling** - Create component registry
3. **Redux Management** - Shared state across components
4. **Vue Integration** - Load React chunks dynamically
5. **Vue-React Interaction** - Bridge communication between frameworks

---

## External Packages

### React App Rewired

Since the React project was bootstrapped with Create React App (CRA), direct modification of webpack configurations wasn't allowed by default.

To customize webpack without ejecting from CRA, the **`react-app-rewired`** npm package was needed, which lets us override the default CRA configuration safely via a `config-overrides.js` file.

```bash
npm install react-app-rewired --save-dev
```

---

## React Build Setup

### Challenge

The React feature that was already in use followed the standard structure, with `index.js` as the entry point and the `App` component as the root. However, for the new React components that needed to be embedded inside the Vue application, the original build — including the full App — was unnecessary.

### Solution

To support this, a separate entry point specifically for the embedded components was needed. For this purpose, we used `react-app-rewired`, which allowed us to override the default Create React App configuration without ejecting.

### Configuration Steps

#### 1. Skip Default Plugins

Since these embedded components didn't require a standalone HTML page, we skipped adding default plugins like `HtmlWebpackPlugin`, which normally generates the `index.html` file during build.

#### 2. Customize Manifest File

The next step involved customizing the manifest file generation:
- Renaming the manifest file (e.g., to `asset-vue-manifest.json`)
- Controlling which files are included in it (e.g., skipping source maps)

The manifest file was used to read all the JS and CSS files generated after the build step.

#### 3. Rename Output Files

The output CSS and JS chunk files were renamed by adding custom prefixes (like `vue-`) and placing them in specific subdirectories (e.g., `static/vue/js/`, `static/vue/css/`) to clearly separate them from the main React build.

#### 4. Update Entry Point

Finally, the entry point of the React app was updated to use a different file, such as `vueIndex.js`, instead of the default `index.js`.

### config-overrides.js

```javascript
const ManifestPlugin = require('webpack-manifest-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = function override(config, env) {
  // Filter out unnecessary plugins
  config.plugins = config.plugins.filter(
    (plugin) => !['HtmlWebpackPlugin', 'ManifestPlugin'].includes(plugin.constructor?.name)
  );

  // Update output file naming
  config.output = {
    ...config.output,
    filename: 'static/vue/vue-[name].[contenthash:8].js',
    chunkFilename: 'static/vue/vue-[name].[contenthash:8].chunk.js'
  };

  // Update CSS extraction plugin
  config.plugins = config.plugins.map(plugin => {
    if (plugin instanceof MiniCssExtractPlugin) {
      return new MiniCssExtractPlugin({
        filename: 'static/vue/css/vue-[name].[contenthash:8].css',
        chunkFilename: 'static/vue/css/vue-[name].[contenthash:8].chunk.css'
      });
    }
    return plugin;
  });

  // Add custom manifest plugin
  config.plugins.push(
    new ManifestPlugin({
      fileName: 'asset-vue-manifest.json',
      generate: (seed, files, entrypoints) => {
        const manifestFiles = files.reduce((manifest, file) => {
          manifest[file.name] = file.path;
          return manifest;
        }, seed);
        
        const entrypointFiles = entrypoints.main
          .filter(fileName => !fileName.endsWith('.map'));
        
        return {
          files: manifestFiles,
          entrypoints: entrypointFiles
        };
      }
    })
  );

  // Update entry point
  config.entry = './src/vueIndex.js';
  
  return config;
};
```

### Build Scripts

#### Original Build Script
```json
"build": "react-scripts build"
```

#### Modified Build Script

Since separate builds are needed as discussed above, a modified version of build is required:

```json
{
  "scripts": {
    "build": "npm run buildVue && react-scripts build",
    "buildVue": "react-app-rewired build && npm run moveBuild",
    "moveBuild": "mv build buildVue"
  }
}
```

#### Build Process

The `buildVue` script performs two key tasks:

1. It builds the app using `react-app-rewired`, placing the output in the `build/` directory
2. It then runs the `moveBuild` script to relocate the generated contents to a separate `buildVue/` directory

This relocation step is necessary because the subsequent `react-scripts build` command will overwrite the `build/` directory. Without moving the Vue build first, its contents would be lost during the React build process.

---

## React Component Manager

After the basic build setup, a component manager was added, which acts as a centralized handler to manage the mounting and updating of React components on demand.

### Exposed Functions

#### mount

Mounts a React component inside a Vue container.

**Parameters:**
- `elementId`: A unique identifier which maps to its React equivalent component
- `containerId`: The parent container ID within which React is to be loaded
- `props`: Props JSON required by the React component
- `eventHandlers`: A JSON object with all event handler callback functions

#### updateProps

Updates props of an already mounted React component.

**Parameters:**
- `elementId`: A unique identifier which maps to its React equivalent component
- `newProps`: The new properties value

### Implementation

```javascript
window.ReactComponentManager = {
  mount: function (elementId, containerId, props, eventHandlers) {
    const container = document.getElementById(containerId);
    const Component = getComponentFromId(elementId);
    
    if (container) {
      ReactDOM.render(
        <Provider store={window.reduxStore}>
          <ReactStateWrapper 
            Component={Component}
            initialProps={props}
            instanceId={elementId}
            eventHandlers={eventHandlers}
          />
        </Provider>,
        container
      );
    }
  },
  
  updateProps: function (elementId, newProps) {
    if (componentInstances[elementId]) {
      componentInstances[elementId](newProps);
    } else {
      console.warn(`Component with ID "${elementId}" not found`);
    }
  }
};
```

### Global Accessibility

This component manager was attached to the `window` object to make it accessible globally. This means we can directly use `window.ReactComponentManager` from Vue.

### Usage Flow

1. When a component is to be loaded on a Vue page, the `mount` function is called with necessary parameters
2. The `mount` function loads a wrapper component which wraps the expected component along with the Redux store
3. On the Vue end, if a property changes, there is no automatic way for the React component to update, so explicit update is needed
4. In such cases, the `updateProps` function is called

---

## React State Wrapper

The wrapper component manages prop updates from external (Vue) sources.

```javascript
const componentInstances = {}; // Stores mounted components

// State wrapper that manages prop changes
class ReactStateWrapper extends React.Component {
  constructor(props) {
    super(props);
    
    // These props are the wrapper which will include the component props
    // When the component needs to be updated from external effect, we need a way to simulate it
    this.state = {
      currentProps: props.initialProps
    };
  }
  
  componentDidMount() {
    // Setting update of the properties and calling it when the component needs explicit update
    componentInstances[this.props.instanceId] = this.setCurrentProps;
  }
  
  componentWillUnmount() {
    delete componentInstances[this.props.instanceId]; // Cleanup
  }
  
  setCurrentProps = (newProps) => {
    this.setState((prevState) => ({
      currentProps: {
        ...prevState.currentProps,
        ...newProps
      }
    }));
  };
  
  render() {
    const { Component, eventHandlers } = this.props;
    return <Component {...this.state.currentProps} eventHandlers={eventHandlers} />;
  }
}
```

### How It Works

- `componentInstances` stores the instance of `setCurrentProps`
- The `updateProps` in `ReactComponentManager` calls this instance-specific `setCurrentProps`
- This updates the React component's state, triggering a re-render

---

## Component Registry

A registry function `getComponentFromId` maps unique identifiers to React components.

```javascript
function getComponentFromId(id) {
  if (id === 'calculator') {
    return Calculator;
  }
  if (id === 'userProfile') {
    return UserProfile;
  }
  // Add more component mappings here
}
```

**Example:** If we have a calculator created in React, we return it when requested with the identifier `'calculator'`.

---

## Redux Store Handling

The Redux store is created as a **singleton instance** and shared across all mounted components.

### Store Creation

```javascript
const Store = createStore(
  rootReducer,
  composeWithDevTools(
    applyMiddleware(thunk)
  )
);

export default Store;
```

### Usage

This store instance is used in:
- The main React app
- All dynamically mounted components

By sharing a single store, all React components can communicate through a unified state management system, even when embedded in different parts of the Vue application.

---

## Vue Integration

After the build, React components can be used like standard JavaScript and CSS files. They can be served and injected into a page using `<script>` and `<link>` tags in `index.html`.

### Dynamic Loading

To dynamically load the necessary JS and CSS chunks, the `asset-vue-manifest.json` file generated during the Vue build is used. This manifest helps identify all relevant output files.

### Loading Assets

```javascript
const path = "/buildVue/asset-vue-manifest.json";
const response = await fetch(path);
const manifest = await response.json();

// Extract JS files
const jsFiles = Object.values(manifest.entrypoints)
  .filter(file => file.match(/static\/flow\/js\/flow-.*\.js$/));

// Extract CSS files
const cssFiles = Object.values(manifest.entrypoints)
  .filter(file => file.match(/static\/flow\/css\/flow-.*\.css$/));
```

### Injecting Assets

All `.js` files were added to `<script>` elements and all `.css` files as `<link rel="stylesheet">` elements.

This allows the React components to be seamlessly loaded and used in any page without requiring a full React app setup.

---

## React-Vue Bridge Component

A Vue wrapper component was implemented that mounts React components inside Vue. This wrapper handled:
- Prop passing from Vue to React
- Event communication using custom events or callbacks

### Implementation

```javascript
Vue.component("reactComponent", {
  props: {
    elementId: String,
    containerId: String,
    reactProps: {
      type: Object,
      default: function() { return {}; }
    },
    eventHandlers: {
      type: Object,
      default: function() { return {}; }
    }
  },
  
  mounted() {
    if (typeof window.ReactComponentManager?.mount !== "function") {
      console.error("ReactComponentManager.mount is not defined");
      return;
    }
    
    if (this.containerId !== null && this.elementId !== null) {
      const container = document.getElementById(this.containerId);
      
      if (!container) {
        console.error(`Container with ID ${this.containerId} does not exist in the DOM.`);
        return;
      }
      
      const current = this;
      window.ReactComponentManager.mount(
        current.elementId,
        current.containerId,
        current.reactProps,
        current.eventHandlers
      );
    }
  },
  
  watch: {
    reactProps(newVal, oldVal) {
      if (this.containerId && this.elementId && 
          JSON.stringify(newVal) !== JSON.stringify(oldVal)) {
        window.ReactComponentManager.updateProps(this.elementId, newVal);
      }
    }
  },
  
  template: `<div :id="containerId"></div>`
});
```

### Usage in Vue Templates

The `reactComponent` can be called in Vue like any other regular Vue component, as all loading and updating logic is handled by the bridge component.

A watcher calls `updateProps` when the props passed to the Vue component update.

```vue
<reactComponent 
  elementId="calculator" 
  containerId="calculatorWrapper" 
  :reactProps="customProps" 
  :eventHandlers="getEventHandler()" 
/>
```

**Parameters:**
- `containerId`: ID of the parent element within which the component is to be loaded
- `elementId`: Unique identifier mapped to the React component
- `reactProps`: Data to pass to the React component
- `eventHandlers`: Event callback functions

### Event Handlers

The `getEventHandler` function returns a JSON object with all major event callbacks. The idea is that React components can have many elements which may need some click functionality.

The click object includes all required click callback functions, meaning any click event happening at the React end that needs an update at the Vue end will call the functions specified accordingly.

```javascript
getEventHandler() {
  const current = this;
  
  return {
    click: {
      onElementClick: (payload) => {
        // Handle click event from React component
        // Update Vue state or trigger Vue actions
      }
    },
    change: {
      onInputChange: (value) => {
        // Handle input change from React
      }
    }
  };
}
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                  Vue Application                     │
│  ┌───────────────────────────────────────────────┐  │
│  │          Vue Components (Vuex)                │  │
│  │                                               │  │
│  │  ┌────────────────────────────────────────┐  │  │
│  │  │    <reactComponent> Bridge             │  │  │
│  │  │                                        │  │  │
│  │  │  ┌──────────────────────────────────┐ │  │  │
│  │  │  │   React Component (Redux)        │ │  │  │
│  │  │  │                                  │ │  │  │
│  │  │  │  - Shared Redux Store            │ │  │  │
│  │  │  │  - Event Handlers                │ │  │  │
│  │  │  │  - Props from Vue                │ │  │  │
│  │  │  └──────────────────────────────────┘ │  │  │
│  │  │                                        │  │  │
│  │  └────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────┘  │
│                                                      │
│  window.ReactComponentManager (Global Bridge)       │
│  window.reduxStore (Shared State)                   │
└──────────────────────────────────────────────────────┘
```

---

## Directory Structure

```
project/
├── src/
│   ├── index.js              # Main React app entry point
│   ├── vueIndex.js           # Vue-embedded React entry point
│   ├── components/
│   │   ├── Calculator.js     # Example React component
│   │   └── ...
│   ├── store/
│   │   └── index.js          # Redux store (singleton)
│   └── reactComponentManager.js
├── config-overrides.js       # Webpack customization
├── build/                    # Main React build output
├── buildVue/                 # Vue-embedded build output
│   ├── asset-vue-manifest.json
│   └── static/
│       ├── flow/vue/
│       │   ├── vue-main.*.js
│       │   └── vue-main.*.css
└── package.json
```

---

## Migration Workflow

### Step 1: Identify Component to Migrate

Choose a small, self-contained component from Vue to migrate first.

### Step 2: Build React Component

Create the React equivalent with the same functionality.

### Step 3: Register Component

Add the component to the registry:

```javascript
function getComponentFromId(id) {
  if (id === 'newComponent') {
    return NewComponent;
  }
  // ... other components
}
```

### Step 4: Embed in Vue

Replace the Vue component with the bridge component:

```vue
<!-- Before: Vue component -->
<old-vue-component :data="componentData" @click="handleClick" />

<!-- After: React component via bridge -->
<reactComponent 
  elementId="newComponent"
  containerId="newComponentWrapper"
  :reactProps="{ data: componentData }"
  :eventHandlers="{ click: { onElementClick: handleClick } }"
/>
```

### Step 5: Test & Iterate

- Verify functionality matches
- Test prop updates
- Test event handling
- Monitor performance

### Step 6: Repeat

Once one component is successfully migrated, repeat the process for the next component.

---

## Best Practices

### ✅ Do's

- Start with small, isolated components
- Maintain a clear component registry
- Use consistent naming conventions (e.g., `vue-` prefix)
- Keep the shared Redux store simple
- Document each migrated component
- Test thoroughly before moving to the next component

### ❌ Don'ts

- Don't migrate tightly coupled components first
- Don't share state directly between Vue and React (use the bridge)
- Don't skip the manifest file - it's crucial for dynamic loading
- Don't forget to clean up component instances on unmount
- Don't migrate too many components simultaneously

---

## Troubleshooting

### React Components Not Loading

**Issue:** React chunks are not being loaded in Vue.

**Solution:** 
- Verify `asset-vue-manifest.json` exists in `buildVue/`
- Check that JS/CSS files match the filter patterns
- Ensure scripts are loaded in the correct order

### Props Not Updating

**Issue:** Vue prop changes don't reflect in React component.

**Solution:**
- Verify the Vue watcher is triggering (add console.log)
- Check that `updateProps` is being called
- Ensure `componentInstances[elementId]` exists

### State Management Conflicts

**Issue:** Redux and Vuex state are out of sync.

**Solution:**
- Don't try to sync Redux and Vuex directly
- Use event handlers to communicate state changes
- Keep React and Vue state separate, only share via props/events

---

## Conclusion

Migrating from Vue to React incrementally can be a practical and low-risk approach, especially for large, complex applications. By bridging the two frameworks carefully — through custom Webpack configuration, shared state management with Redux, and a communication wrapper — React components can be embedded inside Vue without disrupting existing features.

While the technical setup requires careful planning (e.g., avoiding default CRA plugins and exposing React components globally), the resulting architecture supports a smooth transition and reuse of components across frameworks.
---

