# webgl-resources

Browser-side JavaScript libraries for NetXMS WebGL views, packaged as a Maven
resource jar (`org.netxms:webgl-resources`).

Currently this bundles **three.js** (with `OrbitControls`), used by the NetXMS
management console's **3D rack view** (`Rack3DWidget`).

## Why this exists

The 3D rack view loads three.js as a **classpath resource** rather than from a
CDN, because the desktop client and many server networks are offline, and the
embedded browser renders the page via `Browser.setText()` (no base URL, so an
external `<script src>` would not resolve).

Keeping the ~600 KB minified library in the main NetXMS source tree is
undesirable, so it lives here and is consumed as a normal Maven dependency. The
jar contributes the resource at `/webgl/three.min.js` on the classpath; any
component that calls `getResourceAsStream("/webgl/three.min.js")` picks it up.

## Contents

```
webgl-resources/
├── pom.xml
└── src/main/resources/
    └── webgl/
        └── three.min.js     # three.js + OrbitControls (concatenated)
```

- **`webgl/three.min.js`** — the three.js UMD build (global `THREE`) with the
  legacy global `OrbitControls` appended.

## Producing `three.min.js`

The bundled file is the pinned **three.js r147** UMD build with OrbitControls
appended. r147 is the last revision that ships **both** the global UMD
`build/three.min.js` and the global (non-module) `examples/js/.../OrbitControls.js`
— which is what the global-`THREE` page in `Rack3DWidget` expects.

> Do **not** upgrade past r147 without adapting the page: `examples/js/` was
> removed in r148, and `build/three.min.js` (the UMD build) was removed in r161.

To (re)generate it:

```bash
cd src/main/resources/webgl

# three.js core (UMD / global THREE)
curl -L -o three.min.js \
  https://cdn.jsdelivr.net/npm/three@0.147.0/build/three.min.js

# OrbitControls (global; registers THREE.OrbitControls) — append into the same file
curl -L https://cdn.jsdelivr.net/npm/three@0.147.0/examples/js/controls/OrbitControls.js \
  >> three.min.js
```

Mirrors if jsDelivr is unavailable: unpkg (`https://unpkg.com/three@0.147.0/...`),
cdnjs (core only, `https://cdnjs.cloudflare.com/ajax/libs/three.js/r147/three.min.js`),
or the GitHub release `r147` (`build/three.min.js` + `examples/js/controls/OrbitControls.js`).

Keep the MIT license banner that three.js includes at the top of the file.

## Build & install

```bash
# Install to the local Maven repository
mvn install

# Deploy to the NetXMS Maven repository (configure distributionManagement in pom.xml first)
mvn deploy
```

This must be available to the reactor **before** the NetXMS console (`nxmc`) is
built — install it once locally for development, or deploy it to the shared
NetXMS Maven repository for CI.

## Consuming it

Add the dependency to `nxmc/pom.xml`:

```xml
<dependency>
   <groupId>org.netxms</groupId>
   <artifactId>webgl-resources</artifactId>
   <version>1.0.0</version>
</dependency>
```

No code change is required in the console: `Rack3DWidget` already reads
`/webgl/three.min.js` from the classpath and inlines it into the WebGL page.
If the resource is missing, the 3D view shows a "Three.js not bundled" message
instead of failing.

## Versioning

The artifact version (`1.0.0`) is independent of the bundled three.js revision.
The three.js revision is recorded in the `threejs.version` property in `pom.xml`
and as a `Bundled-ThreeJS-Version` entry in the jar manifest. When you change the
bundled library, bump both the property and the artifact version, then update the
dependency version in `nxmc/pom.xml`.

## License

- This repository's packaging/build files: **MIT License**
- The bundled JavaScript (**three.js**, including `OrbitControls`) is licensed
  under the **MIT License** — © three.js authors. MIT is GPL-compatible; the
  license banner is retained inside `three.min.js`. See
  https://github.com/mrdoob/three.js/blob/dev/LICENSE.
