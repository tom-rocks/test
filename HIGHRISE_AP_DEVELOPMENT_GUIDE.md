# Highrise Admin Panel App Development Guide
*Complete guide for building single-spa microfrontend apps for the Highrise Admin Panel*

---

## 📋 Table of Contents
- [Overview](#overview)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Authentication](#authentication)
- [API Integration](#api-integration)
- [Styling & UI](#styling--ui)
- [Building & Deployment](#building--deployment)
- [Common Issues & Solutions](#common-issues--solutions)
- [Code Examples](#code-examples)
- [Best Practices](#best-practices)

---

## Overview

### What are Highrise AP Apps?
Highrise Admin Panel apps are **single-spa microfrontend applications** that integrate directly into the Highrise admin panel. They allow developers to create custom tools for managing game content like items, characters, and other game assets.

### Key Features:
- **Automatic Authentication**: Apps inherit Google OAuth session from the admin panel
- **Direct API Access**: Use the same API endpoints as the admin panel
- **Seamless Integration**: Apps appear as native features within the admin panel
- **React-based**: Built with React 19 and modern tooling

### Architecture:
- **Single-spa**: Microfrontend framework for dynamic module loading
- **SystemJS**: Module format for runtime loading
- **React 19**: Latest React with hooks
- **Mantine UI**: Professional component library
- **React Query**: Data fetching and caching

---

## Getting Started

### 1. Template Structure (from Maciej)
```bash
single-spa/
├── src/
│   ├── api/                  # API layer
│   │   ├── items.model.js    # Type definitions
│   │   ├── items.service.js  # API calls
│   │   ├── items.query.js    # React Query hooks
│   │   └── items.schema.js   # Validation
│   ├── components/           # React components
│   │   └── ItemViewer.jsx    # Main component
│   ├── pw-app.js            # Single-spa entry
│   ├── root.component.js    # Root React component
│   └── theme.js             # Mantine theme
├── .ai                      # AI instructions (guides AI tools)
├── babel.config.json        # Babel config
├── jest.config.js          # Jest config
├── package.json            # Dependencies
└── webpack.config.js       # Webpack config (has ESM issues)
```

### 2. Installation
```bash
# Install dependencies
npm install

# Start development (Note: has issues, see solutions)
npm run start:standalone

# Build for production
npm run build
```

### 3. Key Dependencies
```json
{
  "@mantine/core": "^8.1.2",      // UI components
  "@mantine/form": "^8.1.2",      // Form handling
  "@mantine/hooks": "^8.1.2",     // Utility hooks
  "@tanstack/react-query": "^5.81.5",  // Data fetching
  "react": "^19.0.0",             // React 19
  "react-dom": "^19.0.0",         // React DOM
  "react-hook-form": "^7.60.0",   // Form state
  "single-spa-react": "^6.0.2",   // Single-spa integration
  "zod": "^3.25.75"               // Schema validation
}
```

---

## Project Structure

### API Layer Organization
```javascript
// items.model.js - Type definitions
export const createGetItemRequest = (dispId) => ({
    _type: "GetItemRequest",
    disp_id: dispId,
});

// items.service.js - API calls
export const fetchItem = async (dispId) => {
    const response = await fetch("/api", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(createGetItemRequest(dispId)),
        mode: 'cors',
        credentials: 'include'  // Critical for auth
    });
    return response.json();
};

// items.query.js - React Query hooks
export const useItemQuery = (dispId) => {
    return useQuery({
        queryKey: ["item", dispId],
        queryFn: () => fetchItem(dispId),
        enabled: !!dispId,
    });
};

// items.schema.js - Validation
export const itemIdSchema = z.string().min(1, "Item ID is required");
```

### Component Structure
```javascript
// Standard component pattern
export const ComponentName = () => {
    // State and hooks
    const [state, setState] = useState();
    
    // Form handling
    const form = useForm({
        validate: zodResolver(schema),
    });
    
    // API queries
    const { data, isLoading, error } = useCustomQuery();
    
    // Render
    return (
        <Container>
            {/* Component content */}
        </Container>
    );
};
```

---

## Authentication

### How It Works
1. **No Manual Tokens**: Apps automatically inherit authentication from the admin panel
2. **Google OAuth**: User logs into admin panel via Google
3. **Session Sharing**: Apps use the same session cookies
4. **Automatic**: No need to handle auth in your app

### Required Fetch Options
```javascript
// ALWAYS include these options for API calls
fetch('/api', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
    },
    body: JSON.stringify(payload),
    mode: 'cors',           // Enable CORS
    credentials: 'include'  // Include cookies - CRITICAL!
})
```

### Common Auth Errors
- **401/403**: Session expired - user needs to re-login to admin panel
- **CORS errors**: Missing `mode` or `credentials` options
- **Redirect to login**: User not authenticated in admin panel
- **Login redirects**: Getting redirected to `auth.life4.highrise.game`

### Google OAuth Flow Requirements
- **Multi-step process**: Email → Password → Account Selection → Potential 2FA
- **Critical**: Must complete FULL OAuth flow including account selection
- **Cookie timing**: Save cookies AFTER complete authentication
- **Session persistence**: Use persistent Chrome user data directory for stable sessions

### Authentication Debug Tips
```javascript
// Test auth endpoint to validate authentication
async function testAuth() {
    try {
        const response = await fetch('/test/auth', {
            credentials: 'include'
        });
        if (response.ok) {
            console.log('Authentication valid');
        } else {
            console.log('Authentication failed:', response.status);
        }
    } catch (error) {
        console.error('Auth test error:', error);
    }
}
```

### Working Authentication Cookies Example
When properly authenticated, browser cookies include:
```json
{
  "_oauth2_proxy": "eyJFbWFpbCI6InRvbUBwb2NrZXR3b3JsZHMuY29tIiwiVXNlciI6IjExNDgxMjEyMzE5MTgxNTc4Mjk1NSJ9|1751035848|vCUe3wuZAzpxKjsAadmAfx1Lkvg=",
  "AIOHTTP_SESSION": "\"gAAAAABoYrdUyhRTxxJiMaEniSNxdYjW9I0KwU1RNeMzz4BGCnuTdQEAlvNoB3JvBd_nmEgzRsUX58dHnGHNxFh4kCECN37i1IWk0dkrpR4M1Ph-V5EWSkGfNpY4XlOCCL1SroUVqp0ej4Da5eGaPX9yUxYtMSeiMf8OihlCfFmyw-yrxdRiNqY=\""
}
```
These are handled automatically when accessing apps through the admin panel.

---

## API Integration

### Highrise API Format
All API calls go to `/api` with a `_type` field:

```javascript
// Request format
POST /api
{
    "_type": "GetItemRequest",
    "disp_id": "item-id-here"
}

// Response format
{
    "item": {
        "_id": "...",
        "disp_id": "...",
        "disp_name": "...",
        // ... item data
    },
    "metadata": {},
    "users": {}
}
```

### Available Request Types

#### 1. GetItemRequest (Single Item)
```json
{
    "_type": "GetItemRequest",
    "disp_id": "item-id-here"
}
```
**Purpose**: Get complete item data for viewing/editing
**Returns**: Full item object with all metadata and settings

#### 2. GetBulkItemsRequest (⚡ FASTEST - 50+ items/second)
```json
{
    "_type": "GetBulkItemsRequest",
    "disp_ids": ["item-id-1", "item-id-2", "item-id-3"]
}
```
**Performance**: 175x faster than browser navigation
**Returns**: Array of items with complete metadata

#### 3. GetItemsRequest (Pagination)
```json
{
    "_type": "GetItemsRequest", 
    "page": 1,
    "page_size": 50
}
```
**Purpose**: List items with pagination
**Performance**: ~200 items in 4 seconds

#### 4. SetItemRequest (Update Item)
```json
{
    "_type": "SetItemRequest",
    "item": {
        // Complete item object with modifications
        "disp_id": "item-id-here",
        "disp_name": "New Display Name",
        // ... ALL fields must be included ...
    },
    "make_item_files": true
}
```
**Critical**: Must send COMPLETE item object, not just changes
**Process**: 
1. First use GetItemRequest to fetch current data
2. Modify the fields you want
3. Send entire object back with SetItemRequest

#### 5. SetAssetStatusRequest
```json
{
    "_type": "SetAssetStatusRequest",
    "item_id": "item-id-here", 
    "status": "released" // or "draft", "pending_qa", etc.
}
```
**Purpose**: Change item asset status

### Failed/Invalid Endpoints
❌ `UpdateItemRequest` - Returns 400 Bad Request  
❌ `SetItemDisplayName` - Returns 400 Bad Request  
❌ `UpdateItem` with fields object - Returns 400 Bad Request  
❌ `ItemUpdate` - Returns 400 Bad Request

### Required Headers
```javascript
// For maximum compatibility, include these headers:
{
    "content-type": "application/json",
    "accept-encoding": "identity",
    "origin": "https://dev-ap.highrise.game",  // Match your environment
    "referer": "https://dev-ap.highrise.game/items/tools/bulk-editor"
}
```

### Environment URLs
- **Dev**: `https://dev-ap.highrise.game`
- **Liveops**: `https://liveops-ap.highrise.game` (contains admin/liveops items)
- **Production**: `https://production-ap.highrise.game` (different item set)

### Public Web APIs (No Auth Required)
- **Production**: `https://webapi.highrise.game/`
- **Liveops**: `https://liveops-webapi.highrise.game/`
- **Limitation**: Does NOT contain admin/liveops items

### Thumbnail URLs
```javascript
// Environment-specific thumbnail URLs:

// Dev environment
`https://dev-ap.highrise.game/avataritem/front/${itemId}.png`
`https://dev-ap.highrise.game/avataritem/back/${itemId}.png`

// Liveops environment
`https://liveops-ap.highrise.game/avataritem/front/${itemId}.png`
`https://liveops-ap.highrise.game/avataritem/back/${itemId}.png`

// Production environment
`https://production-ap.highrise.game/avataritem/front/${itemId}.png`
`https://production-ap.highrise.game/avataritem/back/${itemId}.png`

// Characteristics:
// - Format: PNG images
// - Size: 50-250KB for valid items
// - Invalid items: Return ~69 byte placeholder
// - Front views: High quality
// - Back views: Often placeholders
```

### Performance Comparisons
| Method | Speed | Use Case |
|--------|-------|----------|
| Browser navigation | 15-30s/item | Manual testing |
| Browser automation | 3.5s/item | Small batches |
| GetBulkItemsRequest | 50+ items/second | Bulk operations |

### Complete Item Update Example
```javascript
// Step 1: Get current item
const getResponse = await fetch('/api', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        "_type": "GetItemRequest",
        "disp_id": "body-pet_snugglins2025hcchooodierainy"
    }),
    mode: 'cors',
    credentials: 'include'
});

const data = await getResponse.json();
const item = data.item;

// Step 2: Modify the item
item.disp_name = "New Name Here";
item.description = "Updated description";

// Step 3: Save the complete modified item
const setResponse = await fetch('/api', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        "_type": "SetItemRequest", 
        "item": item,  // Complete object required
        "make_item_files": true
    }),
    mode: 'cors',
    credentials: 'include'
});
```

### Image Validation Helper
```javascript
function isValidImage(content, contentType = "") {
    if (!content || content.length < 100) {
        return false;
    }
    
    // Check content type or file signatures
    return (
        contentType.toLowerCase().startsWith('image/') ||
        content.startsWith(b'\x89PNG') ||      // PNG signature
        content.startsWith(b'\xff\xd8\xff') || // JPEG signature  
        content.startsWith(b'GIF8')            // GIF signature
    );
}
```

### API Response Validation
```javascript
// Always validate responses before using
function isValidItemResponse(response) {
    return response && 
           response.item && 
           typeof response.item === 'object' &&
           response.item.disp_id;
}

// Usage
const data = await response.json();
if (!isValidItemResponse(data)) {
    throw new Error('Invalid API response format');
}
```

### Successful API Response Examples

#### GetItemRequest Response
```json
{
    "item": {
        "_id": "6861908fb31cf07a9800c43b",
        "_type": "DContainerArchetype", 
        "disp_id": "cn-6831fa7e179019f7e553bd8b",
        "disp_name": "Buzz Buddy Plush",
        "rarity": "legendary",
        "category": "container",
        "container_type": "gift",
        "created_at": "2025-05-24T16:57:34.133000+00:00",
        "description_key": "<#fdae25>Makes it possible for <b>50 native bees</b> to thrive for a year...",
        "cost": 5000,
        "acquisition_amount": 1,
        "acquisition_cost": 5000,
        "acquisition_currency": "gems",
        "auto_use": false,
        "is_purchasable": true,
        "is_tradable": false,
        "is_vip_only": false,
        "m_exclude": true,
        "m_exclude_marketplace": false,
        "m_exclude_marketplace_sale": false,
        "m_sort_priority": 0,
        "pops_sale_price": 0,
        "probability": 10000,
        "denominator": 10000,
        "sale_price": null,
        "thumbnail_url": "https://cdn.highrisegame.com/container/cn-6831fa7e179019f7e553bd8b/full/",
        "rewards": [
            {
                "reward": {
                    "category": "outfit",
                    "amount": 1,
                    "item_id": "handbag-n_beeitems2025beeplushe"
                },
                "_id": "6831fc81179019f7e553bd92",
                "account_bound": false,
                "metadata": null,
                "descriptors": null
            }
        ]
    },
    "metadata": {},
    "users": {},
    "read_only": false
}
```

#### GetBulkItemsRequest Response
```json
{
    "items": [
        { /* item 1 data */ },
        { /* item 2 data */ },
        { /* item 3 data */ }
    ],
    "metadata": {},
    "users": {}
}
```

### Environment-Specific Items
- **Liveops Environment**: Contains all admin/liveops items for testing
- **Production Environment**: Contains live game items
- **Dev Environment**: May have test items or subset of production
- **Note**: An item existing in one environment doesn't guarantee it exists in another

---

## Styling & UI

### Critical: Import Mantine CSS
```javascript
// In root.component.js - THIS IS REQUIRED!
import "@mantine/core/styles.css";
```

### Custom Theme Setup
```javascript
// theme.js
import { createTheme } from "@mantine/core";

export const theme = createTheme({
    defaultRadius: "md",
    primaryColor: "primary",
    colors: {
        primary: ["#EBF1FD", "#D7E3FC", ...],
        rarities: ["#615F71", "#00B072", "#008BE8", "#9374FD", "#FCBE23", "#54497F"],
    },
    shadows: {
        xs: "0px 3px 4px -5px rgba(24, 24, 28, 0.03)",
        sm: "0px 0px 2px rgba(24, 24, 28, 0.02)",
    },
    components: {
        Button: { /* custom button styles */ },
        Card: { /* custom card styles */ },
        // ... more component customizations
    }
});
```

### Applying Theme
```javascript
// In root component
import { MantineProvider } from "@mantine/core";
import { theme } from "./theme.js";

<MantineProvider theme={theme}>
    <YourApp />
</MantineProvider>
```

### Rarity Colors
```javascript
const getRarityColor = (rarity) => {
    const colors = {
        legendary: 'yellow',
        epic: 'violet',
        rare: 'blue',
        uncommon: 'teal',
        common: 'gray'
    };
    return colors[rarity] || 'gray';
};
```

---

## Building & Deployment

### Working Build Process

1. **Create simplified webpack config** (due to ESM issues):
```javascript
// webpack.config.simple.js
const path = require('path');

module.exports = {
  entry: './src/pw-app.js',
  output: {
    filename: 'index.js',
    path: path.resolve(__dirname, 'dist'),
    libraryTarget: 'system',  // Important for single-spa
  },
  module: {
    rules: [
      {
        test: /\.(js|jsx)$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
        },
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'],
      },
    ],
  },
  resolve: {
    extensions: ['.js', '.jsx'],
  },
  externals: [],  // Bundle everything
};
```

2. **Install CSS loaders** (if not present):
```bash
npm install style-loader css-loader --save-dev
```

3. **Build the app**:
```bash
npx webpack --config webpack.config.simple.js --mode=production
```

4. **Upload Process**:
   - Build creates `dist/index.js`
   - Go to `https://dev-ap.highrise.game/apps`
   - Upload the `index.js` file
   - Give it a meaningful name
   - Ignore drag-drop errors (known issue)
   - Refresh to see your app

### File Size Note
- Apps typically build to 300-600KB
- This includes all dependencies
- Single-spa requires bundling everything

---

## Common Issues & Solutions

### 1. Webpack ESM Module Error
**Error**: `ERR_REQUIRE_ESM` when running webpack
**Solution**: Use simplified webpack config (see above)

### 2. Missing Styles
**Error**: App looks unstyled
**Solution**: Add `import "@mantine/core/styles.css"` to root component

### 3. CORS/Authentication Errors
**Error**: 401/403 or CORS errors
**Solution**: 
- Ensure `credentials: 'include'` in fetch
- User must be logged into admin panel
- Access app through admin panel, not directly

### 4. "Method Not Allowed" (405)
**Error**: 405 on API calls
**Solution**: 
- Check you're using POST (not GET)
- Verify endpoint accepts the request type
- Some endpoints may have different requirements

### 5. Thumbnails Not Loading
**Error**: Images return HTML instead of image
**Solution**: 
- Not all items have thumbnails (expected)
- Container items typically don't have images
- Implement proper error handling for missing images

### 6. Dev Server Not Working
**Error**: `npm run start:standalone` fails
**Solution**: 
- Known issue with template
- Build and upload for testing instead
- Or fix webpack config for local development

### 7. Environment Confusion
**Symptoms**: "Item not found" errors for valid items
**Solutions**:
- Verify target environment (liveops vs production vs dev)
- Check if item exists in intended environment
- Use correct base URL for all requests
- Remember: Liveops has different items than production

### 8. Debug Logging
Enable detailed logging to track:
```javascript
// Add to your API calls for debugging
console.log('Request:', {
    url: '/api',
    payload: requestPayload,
    headers: response.headers
});
console.log('Response status:', response.status);
console.log('Response length:', responseText.length);
```

### Development Testing Endpoints
- **Test auth endpoint**: `GET /test/auth` - Validates authentication
- **Browser service**: Port 5001 for manual authentication when needed  
- **Main interface**: Port 5002 for production web interface

---

## Code Examples

### Complete Image Component with Error Handling
```javascript
function ItemImage({ src, alt, itemId, view }) {
  const [hasError, setHasError] = useState(false);
  const [isLoading, setIsLoading] = useState(true);

  React.useEffect(() => {
    setHasError(false);
    setIsLoading(true);
    
    fetch(src, { credentials: 'include' })
      .then(response => {
        const contentType = response.headers.get('content-type');
        if (!contentType || !contentType.startsWith('image/')) {
          setHasError(true);
        }
        setIsLoading(false);
      })
      .catch(() => {
        setHasError(true);
        setIsLoading(false);
      });
  }, [src, itemId, view]);

  if (hasError) {
    return (
      <Box style={{ 
        height: 200, 
        display: 'flex', 
        alignItems: 'center', 
        justifyContent: 'center',
        backgroundColor: '#f8f9fa',
        borderRadius: '4px'
      }}>
        <Text color="dimmed" size="sm">No {view} image</Text>
      </Box>
    );
  }

  return (
    <Image
      src={src}
      alt={alt}
      withPlaceholder
      placeholder={isLoading ? <Loader size="sm" /> : null}
      height={200}
      fit="contain"
      radius="sm"
      onError={() => setHasError(true)}
    />
  );
}
```

### Form with Validation
```javascript
const schema = z.object({
  dispId: z.string().min(1, "Display ID is required"),
});

const form = useForm({
  initialValues: { dispId: "" },
  validate: zodResolver(schema),
});

<form onSubmit={form.onSubmit(handleSubmit)}>
  <TextInput
    label="Item ID"
    placeholder="e.g., body-m_pants_v1whitepants"
    {...form.getInputProps("dispId")}
    required
  />
  <Button type="submit" loading={isLoading}>
    Fetch Item
  </Button>
</form>
```

### API Service Pattern
```javascript
const makeApiRequest = async (endpoint, payload) => {
  try {
    const response = await fetch(`/api${endpoint}`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload),
      mode: 'cors',
      credentials: 'include'
    });

    if (!response.ok) {
      throw new Error(`API request failed: ${response.status}`);
    }

    return response.json();
  } catch (error) {
    console.error("API request error:", error);
    throw error;
  }
};
```

---

## Best Practices

### 1. Error Handling
- Always provide user-friendly error messages
- Log detailed errors to console for debugging
- Handle loading states with proper UI feedback
- Implement retry logic for transient failures

### 2. Performance
- Use React Query for caching and data management
- Implement proper memoization where needed
- Bundle all dependencies (required for single-spa)
- Optimize images and assets

### 3. Development Workflow
1. Use Maciej's structure as foundation
2. Build frequently to test in actual environment
3. Check browser console for detailed errors
4. Test with various item types (containers, clothing, etc.)

### 4. Code Organization
- Keep API logic in `api/` folder
- Separate concerns (models, services, queries)
- Use TypeScript patterns even in JavaScript
- Follow React best practices

### 5. UI/UX Guidelines
- Use Mantine components consistently
- Follow the custom theme patterns
- Provide clear feedback for all actions
- Handle edge cases gracefully

### 6. Testing Different Items
- Containers: `cn-6831fa7e179019f7e553bd8b` (usually no images)
- Clothing: `body-m_pants_v1whitepants` (usually has images)
- Test with various item types to ensure robustness

---

## Quick Reference

### Essential Imports
```javascript
import React from "react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { MantineProvider } from "@mantine/core";
import "@mantine/core/styles.css";  // CRITICAL!
```

### API Call Template
```javascript
const response = await fetch('/api', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ _type: 'RequestType', ...params }),
  mode: 'cors',
  credentials: 'include'
});
```

### Build Command
```bash
npx webpack --config webpack.config.simple.js --mode=production
```

### Upload URL
```
https://dev-ap.highrise.game/apps
```

---

## Additional Resources

### From Maciej's Instructions
- Start simple with fetching item by ID
- Build incrementally
- Upload and test frequently
- Ping Maciej if app doesn't appear after upload

### Community Resources
- Highrise API documentation (when available)
- Single-spa documentation: https://single-spa.js.org/
- Mantine documentation: https://mantine.dev/

---

*Last updated: January 2025*
*Based on real implementation experience and Maciej's guidance* 