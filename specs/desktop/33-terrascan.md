# TerraScan Desktop — 5-Framework Comparison

## Desktop Rationale

- **GPU-Accelerated Image Processing:** Satellite imagery pipelines process multi-gigapixel rasters requiring GPU compute shaders for radiometric correction, pan-sharpening, band math, and orthorectification — operations that exceed WebGL texture size limits and demand direct Vulkan/Metal/CUDA access. A single Sentinel-2 scene at 10m resolution contains 10,980x10,980 pixels across 13 spectral bands, totaling over 1.5 billion pixel values that must be processed with per-pixel mathematical operations for any useful analysis.

- **Large GeoTIFF Rendering:** Single Sentinel-2 scenes are 10,000x10,000 pixels per band with 13 spectral bands — rendering these as tiled, multi-resolution overlays requires native memory-mapped file access and GPU texture streaming that browsers cannot provide. A typical analyst workspace contains dozens of overlapping scenes requiring a tile pyramid with hundreds of thousands of tiles cached locally and streamed to the GPU on demand during pan and zoom operations.

- **Local ML Inference for Object Detection:** Running YOLO, Mask R-CNN, or custom segmentation models on satellite tiles for building detection, vehicle counting, or land cover classification requires GPU inference with ONNX Runtime or TensorRT — latency and privacy preclude cloud-only approaches. Defense and intelligence analysts require sub-second inference latency across thousands of tiles, and many organizations prohibit transmitting imagery to external cloud services.

- **Offline Map Viewing:** Field analysts, defense users, and disaster response teams operate in disconnected environments — all imagery, basemaps, vector overlays, and ML models must be available locally with no internet dependency. Humanitarian response teams deploying to disaster zones need pre-cached satellite imagery and basemaps for areas where communications infrastructure is destroyed.

- **Multi-Spectral Analysis:** Combining visible, near-infrared, SWIR, and thermal bands for vegetation indices (NDVI), water detection (NDWI), and change detection requires per-pixel band math across large rasters with GPU-parallel computation. Precision agriculture workflows compute NDVI across entire farm boundaries at 10m resolution, requiring billions of per-pixel calculations that only GPU compute can execute interactively.

---

## Desktop-Specific Features

The following features are only feasible on desktop due to GPU raster processing, large imagery datasets, offline requirements, and ML inference needs:

- GPU-accelerated raster processing: radiometric correction, atmospheric compensation (DOS1, 6S), pan-sharpening (Brovey, Gram-Schmidt), and orthorectification via wgpu compute shaders
- Tiled multi-resolution GeoTIFF renderer with GPU texture streaming, LRU tile cache, and smooth zoom transitions for interactive exploration of multi-gigapixel satellite imagery
- Local ONNX Runtime GPU inference for object detection (buildings, vehicles, ships, aircraft), land cover segmentation (urban/forest/water/agriculture), and change detection between temporal pairs
- Offline map tile cache with MBTiles storage for basemap layers (OpenStreetMap, terrain hillshade, administrative boundaries) covering analyst areas of interest at configurable zoom levels
- Multi-spectral band combination viewer with real-time GPU band math: NDVI, NDWI, SAVI, NBR, EVI, and arbitrary user-defined index formulas with instant preview
- Native GDAL integration for reading 100+ raster formats: GeoTIFF, JP2, HDF5, NetCDF, Sentinel SAFE, Landsat MTL, DTED, MrSID, ECW
- Multi-monitor support: imagery viewer on primary display, spectral profile plots and detection result tables on secondary display for efficient analyst workflows
- Background processing queue for batch orthorectification, large-area mosaic generation, and ML inference across thousands of tiles in an AOI
- Vector overlay engine for GeoJSON/Shapefile/GeoPackage with attribute-based styling, labeling, spatial query (point-in-polygon, buffer, intersect), and feature editing
- Time-series animation player for visualizing temporal changes across monthly/yearly satellite imagery stacks with synchronized band combinations and detection overlays

---

## Shared Packages

All shared packages are compiled as Rust libraries with C-compatible FFI exports, enabling consumption from any framework.

### core (Rust)

- Raster processing engine: radiometric calibration (DN to radiance/reflectance), atmospheric correction (DOS1 dark object subtraction, 6S look-up table interpolation), pan-sharpening (Brovey, Gram-Schmidt, PCA), orthorectification with RPC/RPB models — all GPU-accelerated via wgpu compute shaders
- Tiled image renderer: multi-resolution tile pyramid generation with GDAL overview support, GPU texture streaming with configurable LRU cache (default 512 MB VRAM), on-the-fly reprojection between CRS (UTM zones, WGS84, Web Mercator, custom projections via PROJ)
- ML inference pipeline: ONNX Runtime bindings with CUDA/DirectML/CoreML execution providers, configurable sliding window tile extraction with overlap and padding, batched GPU inference, non-maximum suppression for detection consolidation, and result georeferencing back to world coordinates
- Spectral analysis library: band math engine supporting arbitrary arithmetic expressions over any combination of bands, vegetation/water/burn/soil index computation (NDVI, NDWI, NBR, SAVI, EVI), spectral unmixing (NNLS endmember extraction), and principal component analysis — all GPU-parallel via compute shaders
- Change detection engine: automatic image co-registration via feature matching (SIFT/ORB), radiometric normalization (histogram matching, pseudo-invariant features), differencing with adaptive thresholding (Otsu, KI), and object-level change classification via local ML model
- Geospatial I/O: GDAL Rust FFI for reading/writing GeoTIFF, JP2, HDF5, NetCDF with block-level access; GeoJSON/Shapefile/GeoPackage vector reader/writer with R-tree spatial indexing; MBTiles reader/writer for offline basemap tile storage

### api-client (Rust)

- Satellite imagery catalog client for searching and downloading from Sentinel Hub (Copernicus), Planet (PlanetScope, SkySat), Maxar (WorldView, GeoEye), and USGS Earth Explorer (Landsat) APIs with AOI, date range, and cloud cover filtering
- Cloud processing offload for large-area mosaic generation, batch ML inference across entire countries, and compute-intensive atmospheric correction exceeding local GPU capacity
- Basemap tile downloader for pre-caching OpenStreetMap, Mapbox, Stamen, and terrain hillshade tiles for offline use in specified AOI bounds at configurable zoom levels (typically z0-z16)
- License validation with offline activation supporting 90-day grace period for extended field deployments, defense SCIFs, and disaster response operations without internet connectivity
- Annotation sync client for sharing ML training labels, detection results, and analyst assessments across distributed team members via encrypted differential sync

### data (SQLite)

- Imagery catalog with R-tree spatial index on geographic footprints, B-tree temporal index on acquisition date, and rich metadata (cloud cover percentage, sun elevation, off-nadir angle, sensor name, resolution, processing level) for fast multi-criteria search across thousands of scenes
- ML model registry tracking model versions, architecture type, input tensor shape, class labels with colors, mAP/F1 performance metrics, and deployment history for each production detection model
- Detection results store with georeferenced bounding boxes and polygons (WKT geometry), confidence scores, class labels, analyst review status (unreviewed/confirmed/rejected), and temporal linkage for tracking persistent objects
- Project workspace database tracking AOI polygon definitions, processing pipeline configurations, band combination presets, export format preferences, and team collaboration settings

---

## Framework Implementations

Each framework is evaluated for its ability to deliver GPU-accelerated raster processing, multi-resolution tile rendering, and real-time ML inference.

### Electron

**Architecture:** Chromium renderer for React-based analyst UI with Mapbox GL JS or deck.gl for 2D map display and satellite tile rendering; WebGL2 custom shaders for band math visualization overlays; Node.js native addons for GDAL raster access and ML inference; Worker threads for background batch processing

**Tech stack:** React + TypeScript, Mapbox GL JS for map rendering, deck.gl for geospatial data overlays (point clouds, heatmaps), Node.js native addons via napi-rs for Rust core, ONNX Runtime Node.js bindings for ML inference, better-sqlite3 with spatialite extension

**Native module needs:** napi-rs bindings to GDAL for raster I/O, ONNX Runtime native addon for GPU inference, node-gdal-async for geospatial operations, sharp for image processing fallback, proj4js for coordinate transformations

**Bundle size:** ~300-400 MB

**Memory:** ~500-850 MB

**Pros:**
- Mapbox GL JS / deck.gl provide production-grade geospatial visualization with WebGL rendering, vector tile support, 3D terrain, and smooth gesture handling
- Largest ecosystem of JavaScript mapping libraries with extensive satellite imagery display capabilities and pre-built layer types
- Cross-platform with consistent map rendering across Windows/macOS/Linux analyst workstations without platform-specific bugs
- Easy integration with web-based geospatial services, WMS/WMTS tile servers, and cloud-hosted imagery catalogs

**Cons:**
- WebGL2 texture size limits (16K x 16K pixels) restrict single-tile resolution — must manually manage tile pyramid and level-of-detail transitions
- Chromium memory overhead (300+ MB baseline) competes with GDAL raster cache and ML model VRAM requirements on memory-constrained field laptops
- No compute shader access from WebGL2 — band math and ML inference must run out-of-process or via native addons with IPC serialization overhead
- Mapbox GL JS tile renderer not designed for scientific multi-spectral visualization — custom layer implementation needed for false-color band combinations

### Tauri

**Architecture:** Rust backend with GDAL FFI for raster I/O, wgpu compute shaders for GPU-accelerated band math and raster processing, ONNX Runtime for ML inference; custom wgpu tile renderer for multi-resolution imagery display with on-the-fly CRS reprojection; WebView frontend with MapLibre GL JS for vector overlays and UI controls; shared memory for zero-copy raster tile transfer from GDAL through processing to display

**Tech stack:** Rust backend with wgpu compute + rendering, SolidJS frontend with MapLibre GL JS for vector layers, GDAL via FFI (gdal crate), ONNX Runtime Rust bindings (ort crate), sqlx for SQLite with R-tree extension

**Native module needs:** wgpu for GPU compute and tile rendering, GDAL C FFI for raster format support, ONNX Runtime C API bindings for ML inference, PROJ via FFI for coordinate transformations

**Bundle size:** ~55-85 MB

**Memory:** ~150-320 MB

**Pros:**
- wgpu compute shaders for band math and raster processing share GPU context with tile renderer — processed imagery displays without buffer copy or transfer
- Rust-native GDAL integration provides direct block-level raster access without Node.js addon overhead, serialization, or IPC boundaries
- Minimal memory footprint leaves maximum RAM for GDAL raster cache, tile pyramid storage, and large GeoTIFF memory mapping on field laptops
- ONNX Runtime Rust bindings run ML inference in-process with GPU execution provider — no IPC latency for real-time object detection as analyst pans

**Cons:**
- Must build custom geospatial tile renderer in wgpu or synchronize wgpu raster layer with MapLibre vector overlay — non-trivial coordinate system alignment
- GDAL Rust FFI bindings (gdal crate) require careful lifetime management for large raster dataset handles and block-level access patterns
- WebView-hosted MapLibre must synchronize viewport state (center, zoom, bearing) with wgpu raster renderer — dual rendering pipeline complexity
- Smaller Rust geospatial ecosystem compared to Python (rasterio, geopandas, shapely, xarray) — fewer ready-made analysis functions

### Flutter Desktop

**Architecture:** Impeller/Skia for analyst UI panels (AOI drawing, detection review, attribute editing); native texture bridge for map viewport rendered via platform GPU API; Dart FFI to Rust geospatial core for raster processing and ML inference; platform channels for GDAL operations and file system access

**Tech stack:** Dart + Flutter, flutter_rust_bridge for geospatial core FFI, flutter_map for basic map display, custom native texture plugin for raster viewport, drift for SQLite with spatialite

**Native module needs:** Rust geospatial core via FFI, GDAL via C FFI, ONNX Runtime via C FFI, platform-specific GPU interop for raster rendering, PROJ via C FFI

**Bundle size:** ~90-130 MB

**Memory:** ~280-450 MB

**Pros:**
- flutter_map provides basic map display with tile loading, marker placement, polygon overlay, and gesture handling out of the box
- Cross-platform UI for analyst tools: AOI polygon drawing, feature attribute editing, detection result review panels with accept/reject workflows
- Hot reload accelerates iteration on complex analyst workflow interfaces, filter panels, and detection result dashboards
- Smooth animations for time-series playback slider and band combination crossfade transitions between spectral views

**Cons:**
- flutter_map lacks advanced geospatial features — no native multi-spectral rendering, GPU band math, custom projection support, or tiled raster overlay
- Native texture bridge for satellite raster viewport adds a frame of latency for interactive panning — noticeable at high zoom levels
- GPU compute for raster processing and ML inference only through Rust FFI — no direct compute shader access from Flutter layer
- Dart memory pressure from large raster tile operations can trigger GC pauses affecting map responsiveness during rapid pan/zoom

### Swift/SwiftUI (macOS)

**Architecture:** SwiftUI for native macOS analyst UI with inspector panels and sidebar navigation; MapKit for basemap display with custom tile overlay for satellite imagery; Metal compute shaders for GPU-accelerated band math and raster processing; Core ML for on-device ML inference with Apple Neural Engine offload; Swift-Rust FFI for GDAL operations and geospatial algorithms

**Tech stack:** SwiftUI + MapKit + Metal, Core ML for object detection models, Metal compute for raster processing, Swift-Rust FFI for GDAL and geospatial core, GRDB/SQLite with R-tree for spatial queries

**Native module needs:** Metal compute shaders (native), Core ML (native — runs on ANE when available), MapKit (native), GDAL via Rust C FFI, PROJ via C FFI

**Bundle size:** ~35-55 MB

**Memory:** ~120-260 MB

**Pros:**
- Metal compute shaders provide highest throughput for raster band math and image processing on Apple Silicon — NDVI across a full Sentinel-2 scene in milliseconds
- Core ML inference on Apple Neural Engine offloads object detection to dedicated hardware without consuming GPU compute budget for raster processing
- MapKit provides native map display with smooth gesture handling, offline tile caching, and optimized rendering for Apple hardware
- Apple Silicon unified memory eliminates GPU-CPU transfer for processed raster tiles — compute shader output feeds directly to display without copy

**Cons:**
- macOS-only — geospatial analysts predominantly use Windows and Linux workstations, especially in defense and government organizations
- MapKit significantly less flexible than Mapbox GL / MapLibre for custom scientific visualization layers, false-color rendering, and band math overlays
- Core ML model conversion from ONNX/PyTorch via coremltools adds friction to ML model deployment pipeline and may lose precision
- No path to Windows/Linux deployment without complete rewrite of rendering, ML inference, and platform integration layers

### Kotlin Multiplatform

**Architecture:** Compose Multiplatform for shared analyst UI with sidebar, toolbar, and panel layout; custom OpenGL map renderer via LWJGL or embedded JxMapViewer2 for basemap display; JNI to Rust geospatial core for raster processing; ONNX Runtime Java API for ML inference; SQLDelight for geospatial data with R-tree extension

**Tech stack:** Kotlin + Compose Multiplatform, LWJGL for OpenGL tile rendering, JNI to Rust core for GDAL and raster processing, ONNX Runtime Java API for ML inference, SQLDelight for spatial database

**Native module needs:** JNI to Rust geospatial engine, JNI to GDAL for raster I/O, ONNX Runtime Java bindings with GPU provider, LWJGL for OpenGL tile rendering context

**Bundle size:** ~140-200 MB

**Memory:** ~350-600 MB

**Pros:**
- GeoTools (Java) is the most comprehensive open-source geospatial library with full OGC standards support (WMS, WFS, WCS, SLD) and 200+ coordinate reference systems
- ONNX Runtime Java API provides straightforward GPU inference integration for object detection models with DirectML/CUDA execution providers
- Cross-platform desktop with shared analyst UI code across Windows/macOS/Linux deployment environments
- Strong ecosystem of Java geospatial libraries: JTS for geometry operations, GeoTools for projections and format support, imageio-ext for geospatial raster codecs

**Cons:**
- JVM memory overhead (250+ MB baseline) severely limits available RAM for GDAL raster cache and large multi-band imagery on field laptops with 16 GB RAM
- JNI boundary adds measurable latency for transferring raster tiles between Rust processing engine and Java renderer — noticeable during rapid pan/zoom
- No GPU compute access from JVM — all band math, atmospheric correction, and raster processing must traverse the JNI boundary to native Rust/GPU code
- GC pauses cause visible map stutter during rapid panning over large multi-gigapixel imagery, especially with many detection result overlays rendered simultaneously

---

## Comparison Matrix

| Dimension | Electron | Tauri | Flutter | Swift | Kotlin MP |
|-----------|----------|-------|---------|-------|-----------|
| Dev effort (weeks) | 14 | 18 | 22 | 16 | 24 |
| Bundle size | 350 MB | 70 MB | 110 MB | 45 MB | 170 MB |
| Memory usage | 680 MB | 230 MB | 360 MB | 190 MB | 480 MB |
| Startup time | 3.5s | 0.9s | 2.1s | 0.5s | 2.9s |
| Native feel | 5/10 | 7/10 | 6/10 | 10/10 | 6/10 |
| Offline capability | 7/10 | 9/10 | 7/10 | 9/10 | 7/10 |
| GPU/compute access | 4/10 | 9/10 | 4/10 | 10/10 | 4/10 |
| Cross-platform | 9/10 | 9/10 | 8/10 | 2/10 | 7/10 |
| Best fit score | 6/10 | 9/10 | 5/10 | 6/10 | 5/10 |

---

## Recommended Framework

**Tauri** is the best fit for TerraScan.

**Rationale:** Satellite imagery analysis demands GPU-accelerated raster processing (band math, atmospheric correction, pan-sharpening) tightly coupled with interactive multi-resolution tile rendering — Tauri's wgpu backend runs compute shaders and the tile renderer in the same GPU context, enabling zero-copy visualization of processed imagery at full resolution without buffer transfer. GDAL integration via Rust FFI provides direct block-level access to 100+ raster formats without Node.js addon overhead, and ONNX Runtime runs ML inference in-process with GPU execution provider for real-time object detection as the analyst pans across the scene. The minimal memory footprint is critical since satellite imagery workflows routinely consume gigabytes of RAM for raster caching, and field deployment laptops have limited resources.

**Runner-up:** Electron — Mapbox GL JS / deck.gl provide the most mature geospatial visualization ecosystem with production-proven tile rendering and vector overlay capabilities that would significantly accelerate initial development, but the WebGL2 texture limitations, Chromium memory overhead, and lack of GPU compute shader access make it unsuitable for production-scale multi-spectral imagery processing and real-time ML inference workflows.

---

## Monetization (Desktop)

- **Free tier:** Single-scene viewer with RGB band combination, basic NDVI computation with default color ramp, up to 5 locally imported scenes, no ML inference capability
- **Pro license ($69/month):** Unlimited scene library, all spectral indices (NDVI, NDWI, NBR, SAVI, EVI, custom), ML object detection with 3 pre-trained models (buildings, vehicles, ships), batch processing, time-series analysis and animation export
- **Enterprise license ($249/seat/month):** Custom ML model deployment with model management dashboard, multi-analyst collaboration with annotation sync, tasking integration (Planet, Maxar), automated monitoring pipelines with scheduled processing, audit logging for defense compliance (ITAR, security classification marking)
- **Imagery credits:** Pay-per-scene for commercial high-resolution satellite imagery (Planet SkySat at $0.50/km2, Maxar WorldView at $2.50/km2) purchased through integrated catalog browser with volume discount tiers
- **Government/defense pricing:** Annual site license with air-gapped deployment support, ITAR-compliant data handling, FedRAMP-aligned architecture documentation, and dedicated support channel with cleared personnel

---

## MVP Scope (Desktop)

**Core MVP (8 weeks):**

| Week | Milestone |
|------|-----------|
| 1-2 | Rust core: GDAL FFI for GeoTIFF and JP2 reading with block-level tiled access and on-demand decompression; coordinate transformation via PROJ (UTM to Web Mercator for display); SQLite schema with R-tree spatial index for imagery catalog, scene metadata storage, and AOI definitions; basic Tauri shell with file import dialog, scene metadata panel, and catalog list view |
| 3-4 | wgpu tile renderer: multi-resolution tile pyramid generation from GDAL overviews with GPU texture streaming and LRU cache (256 MB default); smooth pan/zoom with inertia and fractional zoom; RGB band combination selector (true color, false color CIR, SWIR); vector overlay engine for GeoJSON polygons with configurable fill/stroke styling and attribute popup |
| 5-6 | GPU raster processing: wgpu compute shaders for real-time band math (NDVI, NDWI, NBR, EVI, custom arithmetic expressions); per-pixel spectral profile display on cursor hover showing all band values; radiometric calibration pipeline (DN to TOA reflectance) for Sentinel-2 L1C and Landsat 8/9 Collection 2; configurable color ramp editor for index visualization (diverging, sequential, categorical) |
| 7 | ML inference: ONNX Runtime integration with GPU execution provider (CUDA on Linux/Windows, CoreML on macOS); pre-trained building detection model (YOLOv8); sliding window inference over visible viewport extent with configurable tile size and overlap; detection result overlay as georeferenced bounding boxes with confidence scores and class labels; result export to GeoJSON with CRS metadata |
| 8 | Offline and workflow: MBTiles basemap tile cache builder for offline background maps (download OSM tiles for AOI at configurable zoom range); scene comparison tools (swipe curtain and side-by-side split viewport modes); basic time-series viewer for multi-date imagery stacks with date picker and animation playback; cross-platform testing (Windows/macOS/Linux) and installer packaging |

**Post-MVP:**

- Atmospheric correction module with DOS1 (dark object subtraction) and 6S radiative transfer look-up tables for surface reflectance computation
- Pan-sharpening (Brovey, Gram-Schmidt, PCA) for resolution enhancement — fuse 10m multi-spectral with 60cm panchromatic imagery
- Change detection engine: bi-temporal co-registration via feature matching, radiometric normalization, differencing with automated Otsu threshold, and change polygon vectorization with area/class statistics
- Custom ML model deployment: user uploads ONNX model with class definitions, input specification, and confidence threshold for domain-specific detection tasks (deforestation, illegal mining, flood extent)
- Automated monitoring: scheduled download of new imagery for defined AOIs from Sentinel Hub, automatic ML inference on new scenes, and desktop/email alert generation when detections exceed threshold
- Multi-scene mosaic generation for seamless coverage with automatic color balancing, seam-line feathering, and cloud masking
- Point cloud / LiDAR viewer for DSM/DTM 3D visualization alongside optical imagery with elevation profile extraction
- Integration with Planet and Maxar tasking APIs for submitting new collection requests with AOI, time window, and off-nadir angle constraints
- Spectral unmixing using non-negative least squares (NNLS) for sub-pixel material composition estimation from multi-spectral data with endmember library
- 3D terrain visualization: satellite imagery draped over DEM surface with exaggerated vertical scale for geomorphological analysis and line-of-sight computation
- Annotation and labeling tools for creating ML training datasets: polygon/rectangle drawing, class assignment, export to COCO/YOLO/Pascal VOC formats
- SAR (Synthetic Aperture Radar) imagery support with speckle filtering, coherence mapping, and interferometric analysis for ground deformation monitoring
- Crop health monitoring dashboard with field boundary management, temporal NDVI tracking, and yield prediction using regression models
- Report generation engine for creating formatted PDF/HTML analyst reports with embedded maps, detection statistics, and temporal comparison charts
- Multi-user access control with role-based permissions (viewer, analyst, admin) and per-project data isolation for organizational deployments
