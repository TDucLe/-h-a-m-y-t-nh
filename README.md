# MEMBER CONTRIBUTION & IMPLEMENTATION DETAILS

**Course:** Computer Graphics — Final Project  
**Project:** The Disco Room  
**Date:** May 2026

---

## 1. Lê Trí Đức (Student ID: 23070399) — The Architect

### 1.1 3D Model Design, Export, and File Loading Pipeline

Lê Trí Đức was responsible for the entire 3D asset pipeline — from designing and exporting 3D models in external tools, to parsing those files at runtime, to uploading geometry onto the GPU. The foundation of this pipeline rests on two fundamental data structures defined in `types.hpp`.

```6:13:discolib/include/discolib/types.hpp
struct Vertex {
 glm::vec3 Position;
 glm::vec3 Normal;
 glm::vec2 TexCoords;
};
```

The `Vertex` struct is the atomic unit of all geometry in the project. Every vertex carries a 3D position, a surface normal for lighting computations, and a 2D texture coordinate for UV mapping.

```15:28:discolib/include/discolib/types.hpp
struct Mesh {
 GLuint VAO = 0;
 GLuint VBO = 0;
 GLuint EBO = 0;
 int vertCount = 0;
 GLenum drawMode = GL_TRIANGLES;

 bool valid() const { return VAO != 0 && vertCount > 0; }
 void draw() const {
 bind();
 if (EBO != 0) glDrawElements(drawMode, vertCount, GL_UNSIGNED_INT, nullptr);
 else glDrawArrays(drawMode, 0, vertCount);
 unbind();
 }
 void destroy() {
 if (VAO) glDeleteVertexArrays(1, &VAO);
 if (VBO) glDeleteBuffers(1, &VBO);
 if (EBO) glDeleteBuffers(1, &EBO);
 VAO = VBO = EBO = 0;
 }
};
```

The `Mesh` struct encapsulates an OpenGL VAO/VBO/EBO trio and provides `draw()` and `destroy()` methods. The design supports both indexed (`glDrawElements`) and non-indexed (`glDrawArrays`) rendering paths.

#### Manual OBJ Loader — `loadOBJ`

The `loadOBJ` function in `loader.cpp` implements a fully hand-written OBJ parser capable of handling all standard OBJ face formats without relying on external parsing libraries.

```52:122:discolib/src/loader.cpp
Mesh loadOBJ(const char* path) {
 std::ifstream file(path);
 if (!file.is_open()) return Mesh{};
 std::vector<glm::vec3> positions;
 std::vector<glm::vec3> normals;
 std::vector<glm::vec2> texcoords;
 std::vector<Vertex> vertices;
 std::vector<unsigned int> indices;

 // Supports v, v/vt, v//vn, v/vt/vn formats
 // Polygons and quads are auto-triangulated via fan triangulation
 // Index deduplication via map: key = "pi/ti/ni" or "pi//ni" or "pi/ti" or "pi"
 // Face normals are averaged when a vertex is shared across multiple faces
 // Returns uploaded Mesh with VAO/VBO ready for glDrawArrays

}
```

Key capabilities include:

- **All face format variants**: `v` (position only), `v/vt` (position + UV), `v//vn` (position + normal), `v/vt/vn` (position + UV + normal).
- **Polygon/quad triangulation**: Fan triangulation converts n-gons into triangles on the fly during face parsing.
- **Index deduplication**: A string-keyed map collapses duplicate vertex entries, accumulating face normals so shared vertices carry averaged normals.
- **Return value**: An uploaded `Mesh` object ready for immediate GPU rendering.

#### Binary STL Loader — `loadSTL`

The `loadSTL` function handles the STL (Stereolithography) binary format, commonly used for CAD-exported geometry.

```127:161:discolib/src/loader.cpp
Mesh loadSTL(const char* path) {
 std::ifstream file(path, std::ios::binary);
 // 80-byte STL header (ignored)
 // 4-byte uint32 triangle count
 // Per triangle (50 bytes):
 //   12 bytes: float nx, ny, nz  (face normal)
 //   36 bytes: 3 × (float x, y, z)  (triangle vertices)
 //   2 bytes: uint16 attribute byte count
 // UV coordinates default to (0, 0)
}
```

The binary format is structured as: 80-byte header, 4-byte triangle count, then 50-byte records per triangle (12-byte normal + 36-byte vertices + 2-byte attribute). UV coordinates default to `(0, 0)` since STL carries no texture data.

#### STL with Planar UV — `loadSTLWithPlanarUV`

```166:210:discolib/src/loader.cpp
Mesh loadSTLWithPlanarUV(const char* path, float scale = 1.0f) {
 // Same binary STL parsing as loadSTL
 // Adds automatic planar UV projection:
 //   - Compute magnitude of face normal on each axis (X, Y, Z)
 //   - Select dominant axis (largest magnitude)
 //   - Project vertices onto dominant plane as UV:
 //     - XY plane dominant → u = x, v = y
 //     - XZ plane dominant → u = x, v = z
 //     - YZ plane dominant → u = y, v = z
}
```

This function extends the basic STL loader by computing a planar UV projection. The dominant axis of the face normal determines which world-plane the UV coordinates are projected onto, ensuring sensible texture mapping for flat surfaces.

### 1.2 Vertex, Mesh Data Structures and GPU Upload

#### Assimp-Based Model Loading — `Model` Class

For more complex models (Furniture, room geometry, NPC characters), the project uses the **Assimp** (Open Asset Import Library) through the `Model` class in `model.cpp`.

```14:24:discolib/include/discolib/model.hpp
struct AABB {
 glm::vec3 min;
 glm::vec3 max;
};

struct MaterialData {
 glm::vec3 diffuseColor = {1.0f, 1.0f, 1.0f};
 GLuint diffuseTexture = 0;
 bool hasTexture = false;
};

struct MeshData {
 std::string name;
 GLuint VAO = 0, VBO = 0, EBO = 0;
 int vertCount = 0;
 int materialIndex = 0;
 glm::vec3 diffuseColor;
 bool hasTexture = false;
 GLuint diffuseTexture = 0;
 std::vector<Vertex> vertices;
};
```

The `Model` class uses `MeshData` to store per-mesh GPU handles and raw vertex arrays, and `MaterialData` to store per-material colors and texture IDs.

```30:79:discolib/src/model.cpp
bool Model::load(const std::string& path) {
 Assimp::Importer importer;
 const aiScene* scene = importer.ReadFile(path,
 aiProcess_Triangulate |
 aiProcess_FlipUVs |
 aiProcess_JoinIdenticalVertices |
 aiProcess_GenNormals |
 aiProcess_CalcTangentSpace);
 // Base directory extracted for relative texture path resolution
 // Delegates to processNode() for hierarchy traversal
 // Calls loadMaterials() for .mtl file parsing
}
```

The Assimp import uses five critical flags:

| Flag | Purpose |
|------|---------|
| `aiProcess_Triangulate` | Convert all faces to triangles |
| `aiProcess_FlipUVs` | Flip V coordinate for OpenGL convention |
| `aiProcess_JoinIdenticalVertices` | Deduplicate vertices across faces |
| `aiProcess_GenNormals` | Generate normals if the file has none |
| `aiProcess_CalcTangentSpace` | Compute tangent vectors for normal mapping |

#### `processMesh` — Naming Convention Routing

```102:250:discolib/src/model.cpp
aiMesh* mesh = scene->mMeshes[i];
// Naming convention routing:
//   COL_*  → collision geometry only (NOT rendered)
//           Special: COL_Pole sets spawn point at center + 3.0 on X
//   TRIGGER_DISCO → NPC position marker (NOT rendered)
//   All other names → rendered geometry
// Builds interleaved vertex buffer:
//   pos(12) + normal(12) + uv(8) + color(12) = 44 bytes/vertex
// Attribute locations: loc=0 (pos), loc=1 (normal), loc=2 (uv), loc=3 (color)
// AABB extracted in local space, transformed to world, padded by 0.05
```

The naming convention system cleanly separates rendering geometry from collision geometry at the mesh level, eliminating the need for a separate collision mesh format.

#### `loadMaterials` — MTL File Parsing

```255:311:discolib/src/model.cpp
// Reads .mtl via Assimp's built-in material reader
// AI_MATKEY_COLOR_DIFFUSE → MTL Kd → fallback diffuseColor
// aiTextureType_DIFFUSE (map_Kd) → primary texture path
// aiTextureType_BASE_COLOR → fallback texture path
// Resolves relative paths by prepending model directory
```

### 1.3 Texture Loading with stb_image

Texture loading is implemented using the **stb_image.h** single-header library, which provides portable image decoding for JPEG, PNG, BMP, TGA, and HDR formats.

```215:238:discolib/src/loader.cpp
GLuint loadTexture(const char* path) {
 int w, h, ch;
 stbi_set_flip_vertically_on_load(true);  // Flip for OpenGL origin
 unsigned char* data = stbi_load(path, &w, &h, &ch, 0);  // Auto channels
 GLuint tex;
 glGenTextures(1, &tex);
 glBindTexture(GL_TEXTURE_2D, tex);
 glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
 glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
 glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
 glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
 glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, w, h, 0, GL_RGBA, GL_UNSIGNED_BYTE, data);
 glGenerateMipmap(GL_TEXTURE_2D);
 stbi_image_free(data);
 return tex;
}
```

Key design decisions:
- **`stbi_set_flip_vertically_on_load(true)`**: Corrects image origin from top-left (image convention) to bottom-left (OpenGL convention).
- **`GL_REPEAT` wrapping**: Allows UV coordinates outside [0,1] to tile textures, essential for floor and wall surfaces.
- **`GL_LINEAR_MIPMAP_LINEAR`**: Trilinear filtering for smooth texture minification.
- **Mipmap generation**: Reduces aliasing artifacts on distant surfaces.

Additionally, `loadCubemapTextures` in `loader.cpp` (lines 243-265) loads six face textures for the skybox cubemap using `GL_TEXTURE_CUBE_MAP` with faces in the standard order: +X, -X, +Y, -Y, +Z, -Z.

### 1.4 Model Matrix Transformations for Scene Composition

In the render loop (primarily `App.cpp`), each object in the scene is placed using model matrices computed from GLM:

```cpp
// Conceptual render loop pattern (App.cpp)
glm::mat4 model = glm::mat4(1.0f);
model = glm::translate(model, glm::vec3(x, y, z));    // Translation
model = glm::scale(model, glm::vec3(sx, sy, sz));     // Scale
// Rotation via glm::rotate(model, angle, axis)
glUniformMatrix4fv(glGetUniformLocation(shader, "uModel"), 1, GL_FALSE, glm::value_ptr(model));
mesh.draw();
```

- **`glm::translate`**: Positions the room walls, floor, furniture, and paintings at their world coordinates.
- **`glm::scale`**: Scales models that were exported at non-world-unit sizes to match the scene scale.
- **`glm::rotate`**: Orients objects (e.g., rotated furniture, tilted paintings).

The model matrix is passed as `uModel` to `SCENE_VS`, where it transforms vertex positions to world space and is combined with `uLightSpaceMatrix` for shadow map rendering:

```92:107:discolib/src/renderer.cpp
void main() {
 vec4 wp = uModel * vec4(aPos, 1.0);
 vFragPos = wp.xyz;
 vNormal = mat3(transpose(inverse(uModel))) * aNormal;
 vTexCoords = aTexCoords;
 vFragPosLS = uLightSpaceMatrix * wp;
 gl_Position = uProjection * uView * wp;
}
```

The normal transformation uses `transpose(inverse(...))` to correctly handle non-uniform scaling and shearing.

---

## 2. Lê Đức Hùng (Student ID: 23070967) — The Core Engineer & Visual Scientist

### 2.1 Lighting System and GLSL Shaders

Lê Đức Hùng implemented the core rendering engine: the Blinn-Phong lighting pipeline, the three-mode disco lighting system, and the shadow mapping subsystem.

#### SCENE_VS — Scene Vertex Shader

```cpp
92:107:discolib/src/renderer.cpp
#version 330 core
layout(location=0) in vec3 aPos;
layout(location=1) in vec3 aNormal;
layout(location=2) in vec2 aTexCoords;
out vec3 vFragPos;
out vec3 vNormal;
out vec2 vTexCoords;
out vec4 vFragPosLS;
uniform mat4 uModel, uView, uProjection, uLightSpaceMatrix;
void main() {
 vec4 wp = uModel * vec4(aPos, 1.0);
 vFragPos = wp.xyz;
 vNormal = mat3(transpose(inverse(uModel))) * aNormal;
 vTexCoords = aTexCoords;
 vFragPosLS = uLightSpaceMatrix * wp;
 gl_Position = uProjection * uView * wp;
}
```

The vertex shader outputs four varyings: world-space fragment position (`vFragPos`), world-space normal (`vNormal`), texture coordinates (`vTexCoords`), and light-space fragment position (`vFragPosLS`). The `transpose(inverse(uModel))` pattern correctly transforms normals under non-uniform scaling.

#### SCENE_FS — Blinn-Phong Lighting with Three Disco Modes

The scene fragment shader implements a comprehensive lighting system with ambient, diffuse, and specular Blinn-Phong components and three distinct disco lighting modes.

**Ambient Component:**
```cpp
// Within SCENE_FS (renderer.cpp lines ~109-126)
vec3 ambient = 0.05 * uLightColor;  // 5% ambient intensity
```

**Diffuse Component:**
```cpp
float diff = max(dot(norm, ldir), 0.0);
vec3 diffuse = diff * lightColor;
```

**Specular (Blinn-Phong Half-Vector):**
```cpp
vec3 viewDir = normalize(uViewPos - vFragPos);
vec3 halfDir = normalize(ldir + viewDir);
float spec = pow(max(dot(norm, halfDir), 0.0), 32.0);
vec3 specular = 0.5 * spec * lightColor;
```

**Three Disco Light Modes:**

| Mode | Name | Beam Count | Color | Volumetric | Special Effect |
|------|------|-----------|-------|------------|----------------|
| 0 | Laser Storm | 50 | Multi-color rotating RGB | Yes (15 steps) | Rainbow per-beam via golden-ratio hash |
| 1 | Slow Dance | 50 | White/Cyan | Yes (15 steps) | 8Hz strobe (square wave) + purple ambient |
| 2 | Spotlight | 8 | Warm red→orange→yellow | No | 8 sweeping focused beams |

The per-beam direction is computed using the **golden-angle spiral** on a unit sphere — a technique that provides near-uniform angular distribution of light beams:

```cpp
// getBeamLight() in SCENE_FS (renderer.cpp lines ~240-260)
float theta = acos(1.0 - 2.0 * (i + 0.5) / float(N));  // polar angle
float phi = 2.39996 * i + sweepOffset;                   // golden angle ≈ 137.5°
vec3 dir = vec3(sin(theta)*cos(phi), -cos(theta), sin(theta)*sin(phi));
```

The **uTime** uniform drives the RGB color cycle for each disco beam:

```cpp
// App.cpp lines 478-493 (time-based hue cycling)
float hue = fmod(time * 0.2f + (float)i * 0.125f, 1.0f);
float h6 = hue * 6.0f;
int hi = int(h6); float hf = h6 - hi;
// Golden-ratio-based hue stepping
switch(hi) {
 case 0: r=1; g=hf; b=0; break;    // Red → Yellow
 case 1: r=1-hf; g=1; b=0; break;  // Yellow → Green
 case 2: r=0; g=1; b=hf; break;    // Green → Cyan
 case 3: r=0; g=1-hf; b=1; break;  // Cyan → Blue
 case 4: r=hf; g=0; b=1; break;     // Blue → Magenta
 case 5: r=1; g=0; b=1-hf; break;  // Magenta → Red
}
```

Flickering is achieved via a sine-based intensity modulation:

```cpp
float flicker = 0.6f + 0.4f * sin(time * 8.0f + (float)i * 1.7f);
```

### 2.2 Shadow Mapping with PCF Anti-Aliasing

#### Depth Pass — `DEPTH_VS` and `DEPTH_FS`

Shadow mapping is implemented as a two-pass technique. The first pass renders depth from the light's perspective using a dedicated depth shader.

```cpp
354:359:discolib/src/renderer.cpp
#version 330 core
layout(location=0) in vec3 aPos;
uniform mat4 uLightSpaceMatrix, uModel;
void main() {
 gl_Position = uLightSpaceMatrix * uModel * vec4(aPos, 1.0);
}
```

```cpp
361:364:discolib/src/renderer.cpp
#version 330 core
void main() { /* depth-only: no color output needed */ }
```

#### Shadow Map Creation — `createShadowMap`

```327:348:discolib/src/loader.cpp
ShadowMap createShadowMap(int res = 2048) {
 // Creates framebuffer object (FBO)
 glGenFramebuffers(1, &fbo);
 glBindFramebuffer(GL_FRAMEBUFFER, fbo);
 // Depth texture: GL_DEPTH_COMPONENT24 for 24-bit depth precision
 glGenTextures(1, &tex);
 glBindTexture(GL_TEXTURE_2D, tex);
 glTexImage2D(GL_TEXTURE_2D, 0, GL_DEPTH_COMPONENT24, res, res, 0,
              GL_DEPTH_COMPONENT, GL_UNSIGNED_INT, nullptr);
 glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
 glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
 glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_BORDER);
 glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_BORDER);
 float borderColor[] = {1.0f, 1.0f, 1.0f, 1.0f};
 glTexParameterfv(GL_TEXTURE_2D, GL_TEXTURE_BORDER_COLOR, borderColor);
 glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_TEXTURE_2D, tex, 0);
 // No color attachment — depth-only pass
}
```

#### `lightSpaceMatrix` Computation

In `App.cpp`, the light space matrix is constructed as the product of an orthographic projection and a look-at view from the light:

```cpp
// App.cpp — moonlight directional light
glm::vec3 moonDir = normalize(glm::vec3(0.3f, 1.0f, 0.2f));
glm::mat4 lProj = glm::ortho(-20.0f, 20.0f, -20.0f, 20.0f, -50.0f, 150.0f);
glm::mat4 lView = glm::lookAt(-moonDir * 50.0f, glm::vec3(0,0,0), glm::vec3(0,1,0));
glm::mat4 lightSpaceMatrix = lProj * lView;
// Passed as uLightSpaceMatrix to both depth pass and scene pass
```

#### `calcShadow` with PCF (Percentage-Closer Filtering)

```cpp
127:141:discolib/src/renderer.cpp
float calcShadow(vec4 fragPosLS) {
 vec3 proj = fragPosLS.xyz / fragPosLS.w;
 proj = proj * 0.5 + 0.5;
 if (proj.x < 0.0 || proj.x > 1.0 || proj.y < 0.0 || proj.y > 1.0) return 0.0;
 float currentDepth = proj.z;
 float bias = max(0.002 * (1.0 - dot(norm, ldir)), 0.0005);
 float shadow = 0.0;
 float sz = 1.0 / float(uShadowMapRes);
 // 3x3 PCF kernel — 9 samples averaged
 for (int x = -1; x <= 1; x++) {
 for (int y = -1; y <= 1; y++) {
 float pcfDepth = texture(uShadowMap, proj.xy + vec2(x,y)*sz).r;
 shadow += (currentDepth - bias > pcfDepth) ? 1.0 : 0.0;
 }
 }
 return shadow / 9.0;
}
```

The PCF technique reduces shadow aliasing by averaging 9 depth comparisons from a 3×3 neighborhood around the fragment's shadow map coordinate. The **bias** term (adaptive: `max(0.002 × (1−N·L), 0.0005)`) prevents "shadow acne" — self-shadowing artifacts caused by insufficient depth precision — while being minimized to avoid "peter-panning" (detached shadows).

### 2.3 Volumetric Raymarching and Cone Beam Effects

#### Volumetric Ray Marching in `SCENE_FS`

The disco scene features volumetric light beams rendered via ray marching from the camera through each fragment toward the light sources:

```cpp
294:308:discolib/src/renderer.cpp
// Inside SCENE_FS main(), after standard Blinn-Phong
if (uLightMode == 0 || uLightMode == 1) {
 vec3 marchDir = normalize(vFragPos - uViewPos);
 float marchLen = length(vFragPos - uViewPos);
 const int STEPS = 15;
 float stepLen = marchLen / float(STEPS);
 for (int s = 1; s <= STEPS; s++) {
 float t = float(s) * stepLen;
 vec3 sampleP = uViewPos + marchDir * t;
 lighting += getBeamLight(sampleP, uLightMode) * 0.07;
 }
 lighting = clamp(lighting, 0.0, 1.0);
}
```

Each step samples the `getBeamLight()` function at an incrementally farther point along the view ray, accumulating scattered light. This creates the characteristic "foggy shaft of light" appearance of volumetric illumination.

#### `getBeamLight` — Cone Beam Intensity Function

```cpp
197:278:discolib/src/renderer.cpp
// Unified beam lighting: computes per-fragment cone beam illumination
// Input: world-space fragment position
// Output: vec3 accumulated beam color+intensity

// Per-beam: direction = goldenAngleSpiral(i)
// Sweep offset = uTime * sweepSpeed (rotating beams)
// Cone intensity = smoothstep(coneWidth, 0, angularDistance)
// Mode 0: laser storm — vibrant multi-color, high intensity
// Mode 1: slow dance — white/cyan, strobing at 8Hz
// Mode 2: spotlight — warm, no volumetric contribution
```

The cone intensity uses a `smoothstep` falloff from the beam axis, creating the characteristic narrow-cone spotlight effect.

#### `VOL_BEAM_VS` / `VOL_BEAM_FS` — Instanced Volumetric Cone Meshes

```cpp
369:439:discolib/src/renderer.cpp
// VOL_BEAM_VS: passes world position + instance color
// VOL_BEAM_FS:
//   - Z-fighting offset: clipPos.z -= 0.001 * clipPos.w
//   - Radial falloff: pow(1.0 - dist, 220.0) — razor-sharp cone edges
//   - White-hot core: exp(-dist * 8.0) additive white glow
//   - Mode 0 alpha: 0.55, Mode 1: 0.18, Mode 2: 0.35
//   - Additive blending: src=GL_SRC_ALPHA, dst=GL_ONE
```

The instanced cone mesh (`createConeMesh` in `loader.cpp` lines 385-413) defines a shared geometry — tip at origin, base at `(0, -coneLength, 0)` with radius 0.12 — reused for all disco beam instances via `glDrawArraysInstanced`.

#### `GODRAY_VS` / `GODRAY_FS` — God Ray Particle System

```cpp
444:486:discolib/src/renderer.cpp
// GodRaySystem: 1000 max point-sprite particles (createGodRaySystem)
// Point size oscillates with sine wave for shimmering effect
// UV from gl_PointCoord for radial glow
// Glow falloff: exp(-dist * 5.0 * vGlow)
// Additive blending for light accumulation
// updateGodRaySystem(): spawns above ball, drifts up + random horizontal,
// fades by lifetime, respawns dead particles
```

### 2.4 Collision Detection: AABB and Hexagonal Room Geometry

#### Hexagonal Room — `outsideHexRoom`

The disco room is a hexagonal prism. Collision uses the **apothem** (distance from center to the midpoint of a side) to test each of the 6 faces independently:

```cpp
28:55:discolib/src/camera.cpp
constexpr float ROOM_CIRCUMRADIUS = 5.5f;     // circumradius of hex room
constexpr float PLAYER_RADIUS = 0.35f;       // player collision radius

inline bool pastFace(float x, float z, float angleDeg, float limit) {
 float rad = radians(angleDeg);
 float nx = -sin(rad), nz = -cos(rad);  // outward-facing unit normal
 return (x * nx + z * nz) > limit;      // signed distance from face
}

inline bool outsideHexRoom(float x, float z) {
 float apothem = ROOM_CIRCUMRADIUS * 0.866025f;  // = 4.763
 float limit = apothem - PLAYER_RADIUS;
 // 6 faces at: 30°, 90°, 150°, 210°, 270°, 330°
 if (pastFace(x, z, 30.0f,  limit)) return true;
 if (pastFace(x, z, 90.0f,  limit)) return true;
 if (pastFace(x, z, 150.0f, limit)) return true;
 if (pastFace(x, z, 210.0f, limit)) return true;
 if (pastFace(x, z, 270.0f, limit)) return true;
 if (pastFace(x, z, 330.0f, limit)) return true;
 return false;
}
```

The formula `ROOM_CIRCUMRADIUS × 0.866025` converts the circumradius (vertex-to-center distance) to the apothem (face-to-center distance) for a regular hexagon.

#### Furniture Collision — `insideFurniture`

```cpp
57:61:discolib/src/camera.cpp
inline bool insideFurniture(float x, float z) {
 // Furniture center: fc = (0, FLOOR_Y + 0.70, 0)
 float dx = abs(x - fc.x);
 float dz = abs(z - fc.z);
 return dx < FURNITURE_HALF_W + PLAYER_RADIUS  // HALF_W = 0.30
     && dz < FURNITURE_HALF_L + PLAYER_RADIUS; // HALF_L = 0.80
}
```

The sofa furniture is modeled as an axis-aligned bounding box in world space, expanded by the player radius.

#### City Scene AABB Collision — `applyCityCollision`

For the city scene, the player collides with AABB obstacles extracted from models (prefixed `COL_*`):

```cpp
110:127:discolib/src/camera.cpp
void CameraController::applyCityCollision(glm::vec3& pos, const glm::vec3& next) const {
 const float COLLISION_SKIN = 0.10f;  // shrink colliders to prevent tunneling
 for (size_t i = 0; i < cityMins.size(); i++) {
 glm::vec3 mn = cityMins[i] + COLLISION_SKIN;
 glm::vec3 mx = cityMins[i] + CITY_HALF_SIZE - COLLISION_SKIN;
 glm::vec3 tryX(next.x, pos.y, pos.z);
 glm::vec3 tryZ(pos.x, pos.y, next.z);
 // Try X movement independently — wall sliding
 bool blockX = (tryX.x + PLAYER_RADIUS < mn.x || tryX.x - PLAYER_RADIUS > mx.x ||
                tryX.z + PLAYER_RADIUS < mn.z || tryX.z - PLAYER_RADIUS > mx.z);
 if (!blockX) pos.x = next.x;
 // Try Z movement independently — wall sliding
 bool blockZ = (pos.x + PLAYER_RADIUS < mn.x || pos.x - PLAYER_RADIUS > mx.x ||
                tryZ.z + PLAYER_RADIUS < mn.z || tryZ.z - PLAYER_RADIUS > mx.z);
 if (!blockZ) pos.z = next.z;
 }
}
```

The **independent X/Z resolution** (trying one axis at a time) enables wall-sliding behavior — the player can glide along walls rather than stopping dead when approaching at an angle.

#### Combined `isBlocked` Function

```cpp
28:95:discolib/src/camera.cpp
// Combines: boundary check + outsideHexRoom() + insideFurniture()
// Returns true if the proposed position collides with any boundary or furniture
inline bool isBlocked(float x, float z) {
 if (outsideHexRoom(x, z)) return true;
 if (insideFurniture(x, z)) return true;
 float BOUNDARY = 2.0f;
 if (x < -BOUNDARY || x > BOUNDARY || z < -BOUNDARY || z > BOUNDARY) return true;
 return false;
}
```

### 2.5 Post-Processing: Fisheye Lens Effect

#### Post-Processing FBO Setup

```cpp
967:1034:discolib/src/renderer.cpp
// Renderer::buildPostProcessShaders()
// 1. Creates FBO with RGBA color texture (same size as screen)
// 2. Creates GL_DEPTH_COMPONENT24 renderbuffer for depth preservation
// 3. Creates screen quad VAO/VBO (fullscreen triangle, NDC coordinates)
// 4. Compiles PP_VS (passthrough) and PP_FS (fisheye/barrel distortion)

GLuint fbo, fboTex, fboDepth;
glGenFramebuffers(1, &fbo);
glBindFramebuffer(GL_FRAMEBUFFER, fbo);
glGenTextures(1, &fboTex);
glBindTexture(GL_TEXTURE_2D, fboTex);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, SCR_W, SCR_H, 0, GL_RGBA,
             GL_UNSIGNED_BYTE, nullptr);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, fboTex, 0);
glGenRenderbuffers(1, &fboDepth);
glBindRenderbuffer(GL_RENDERBUFFER, fboDepth);
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT24, SCR_W, SCR_H);
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_RENDERBUFFER, fboDepth);
```

#### Fisheye / Barrel Distortion — `PP_FS`

```cpp
926:965:discolib/src/renderer.cpp
#version 330 core
in vec2 vUV; out vec4 FragColor;
uniform sampler2D uScene;
uniform float uAspectRatio;
void main() {
 vec2 uv = vUV;
 vec2 p = (uv - 0.5) * 2.0;
 p.x *= uAspectRatio;
 float r = length(p);
 const float DISTORTION = 0.15;
 float k = 1.0 + DISTORTION * r * r;  // barrel distortion polynomial
 p *= k;
 p.x /= uAspectRatio;
 vec2 dUV = p * 0.5 + 0.5;
 if (dUV.x < 0.0 || dUV.x > 1.0 || dUV.y < 0.0 || dUV.y > 1.0)
 FragColor = vec4(0.0, 0.0, 0.0, 1.0);
 else
 FragColor = texture(uScene, dUV);
}
```

The fisheye effect uses a **barrel distortion** polynomial `k = 1 + 0.15r²`. The distortion is radially symmetric — center pixels are unchanged, edge pixels are pushed outward, creating the characteristic wide-angle lens bulge.

The post-process is executed as a **separate render pass**: the scene is first rendered to the FBO, then the FBO texture is drawn to the screen via the post-process quad.

---

## 3. Vũ Hữu Trí (Student ID: 23070745) — The Interactive Engineer

### 3.1 Camera System and Mathematical Foundations

Vũ Hữu Trí implemented the full interactive camera system: the mathematical camera model, the mouse look pipeline, the WASD movement controller, and the dual-mode camera (FPS/CCTV).

#### `Camera` Struct and `CameraMode` Enum

```7:15:discolib/include/discolib/camera.hpp
enum class CameraMode { FPS, CCTV };

struct Camera {
 glm::vec3 pos   = glm::vec3(0.0f, 2.95f, 3.0f);
 float yaw   = -90.0f;   // horizontal rotation (degrees)
 float pitch = 0.0f;     // vertical rotation (degrees, clamped ±89°)
 CameraMode mode = CameraMode::FPS;
 void applyMouse(float dx, float dy, float sens);
 glm::mat4 viewMatrix() const;
 glm::vec3 eyePosition() const;
};
```

#### `Camera::viewMatrix()` — View Matrix via `glm::lookAt`

```67:74:discolib/src/camera.cpp
glm::mat4 Camera::viewMatrix() const {
 glm::vec3 fwd(cos(yaw)*cos(pitch), sin(pitch), sin(yaw)*cos(pitch));
 return glm::lookAt(pos, pos + normalize(fwd), glm::vec3(0, 1, 0));
}
```

The forward direction vector is computed from spherical coordinates:
- **Yaw** (horizontal): `fx = cos(yaw)`, `fz = sin(yaw)`
- **Pitch** (vertical): `fy = sin(pitch)`, `fx *= cos(pitch)`, `fz *= cos(pitch)`

The up vector is fixed at `(0, 1, 0)`. `glm::lookAt` constructs the view matrix that transforms world coordinates into the camera's view space.

#### `Camera::applyMouse` — Pitch Clamping (Gimbal Lock Prevention)

```62:65:discolib/src/camera.cpp
void Camera::applyMouse(float dx, float dy, float sens) {
 yaw += dx * sens;
 pitch = glm::clamp(pitch + dy * sens, -89.0f, 89.0f);
}
```

Pitch is clamped to `[−89°, +89°]` to prevent gimbal lock — a singularity where the forward and up vectors become collinear, causing the cross product to produce an undefined right vector.

### 3.2 Input Processing: Mouse Callback and WASD Movement

#### `mouseCallback` — Raw Mouse Input

```cpp
165:173:discolib/src/App.cpp
static void mouseCallback(GLFWwindow* win, double xpos, double ypos) {
 static bool firstMouse = true;
 static float lastX = 0, lastY = 0;
 constexpr float MOUSE_SENS = 0.10f;
 if (firstMouse) { lastX = (float)xpos; lastY = (float)ypos; firstMouse = false; return; }
 float dx = (float)xpos - lastX;
 float dy = (float)ypos - lastY;
 lastX = (float)xpos; lastY = (float)ypos;
 camCtrl.onMouseMove(dx, dy, MOUSE_SENS);  // delegates to Camera::applyMouse
}
```

The first-frame guard (`firstMouse`) prevents camera snapping on initial mouse capture. Sensitivity of `0.10` maps pixel displacement to degrees of rotation.

#### `CameraController::processInput` — WASD with Collision

```cpp
129:194:discolib/src/camera.cpp
void CameraController::processInput(GLFWwindow* win, float dt) {
 // V key: toggle noclip (2.5× speed)
 if (glfwGetKey(win, GLFW_KEY_V) == GLFW_PRESS) toggleNoclip();

 float speed = noclip ? CITY_SPEED * 2.5f : CITY_SPEED;
 glm::vec3 fwd(cos(yaw), 0, sin(yaw));  // horizontal forward
 glm::vec3 rgt = normalize(cross(fwd, glm::vec3(0,1,0)));
 glm::vec3 next = pos;
 if (glfwGetKey(win, GLFW_KEY_W) == GLFW_PRESS) next += fwd * speed * dt;
 if (glfwGetKey(win, GLFW_KEY_S) == GLFW_PRESS) next -= fwd * speed * dt;
 if (glfwGetKey(win, GLFW_KEY_A) == GLFW_PRESS) next -= rgt * speed * dt;
 if (glfwGetKey(win, GLFW_KEY_D) == GLFW_PRESS) next += rgt * speed * dt;
 // Noclip: no collision, direct position update
 // Normal mode: applyCityCollision (independent X/Z wall-sliding)
 // Disco scene: isBlocked() check → no movement if blocked
}
```

Key design decisions:
- **Yaw-only movement**: `fwd.y = 0` ensures the player always moves on the horizontal plane, regardless of camera pitch.
- **Right vector via cross product**: `cross(fwd, up)` produces the correct strafe direction perpendicular to forward.
- **Noclip mode**: `V` key toggles 2.5× speed flying mode with no collision checks.
- **Wall sliding**: In the city scene, `applyCityCollision` tries X and Z independently so the player can slide along walls.

### 3.3 State Machine and Camera Mode Switching (FPS ↔ CCTV)

#### Game and Scene State Enums

The application uses two independent state machines:

```cpp
// App.cpp
enum class SceneState { CITY_SCENE, DISCO_SCENE };
enum class GameState  { STATE_EXPLORING, STATE_TALKING, STATE_DIALOGUE };
```

| State | Description |
|-------|-------------|
| `CITY_SCENE` | Outdoor city environment with moonlight and shadow mapping |
| `DISCO_SCENE` | Indoor hexagonal disco room with volumetric light beams |
| `STATE_EXPLORING` | Normal FPS or CCTV camera, WASD movement active |
| `STATE_TALKING` | NPC dialogue active, camera follows NPC |
| `STATE_DIALOGUE` | Console menu displayed (Enter Disco / Stay in City), input paused |

#### CCTV Camera Implementation

```cpp
396:411:discolib/src/App.cpp
if (playerCam.mode == CameraMode::CCTV) {
 // Fixed overhead position above the disco floor
 glm::vec3 cctvPos = glm::vec3(DISCO_CCTV_POS.x, orbitHeight, DISCO_CCTV_POS.z);
 // orbitHeight = FLOOR_Y_DISCO + 3.2f = 1.0f + 3.2f = 4.2f
 // DISCO_CCTV_POS = (0, 4.2f, 4.7f)
 // User retains full mouse look (yaw/pitch) from this fixed position
 glm::vec3 fwd(cos(yaw)*cos(pitch), sin(pitch), sin(yaw)*cos(pitch));
 view = glm::lookAt(cctvPos, cctvPos + normalize(fwd), glm::vec3(0,1,0));
}
```

The CCTV camera is a **fixed-position, free-look** camera. The eye position is locked at `(0, 4.2, 4.7)` — above the disco floor at eye level of ~1.7m above the dance floor. The user retains mouse-controlled yaw/pitch rotation, but the position cannot be moved with WASD.

### 3.4 Portal Transition System and Interactive Dialogue UI

#### Portal Positions

```cpp
// App.cpp constants
constexpr glm::vec3 DISCO_PORTAL_POS = glm::vec3(-0.4f, FLOOR_Y, 8.2f);
// FLOOR_Y = 1.0f (disco floor at y = 1.0)
// City NPC position: extracted from TRIGGER_DISCO mesh center
```

#### Transition Triggers — `E` Key and Key 1/2

```cpp
837:868:discolib/src/App.cpp
// In key callback (GLFW_KEY_E pressed):
if (gameState == STATE_EXPLORING) {
 // Check proximity to NPC (city scene)
 float distNPC = length(playerCam.pos - npcWorldPos);
 if (distNPC < 2.5f) {
  gameState = STATE_DIALOGUE;  // Show console menu
 }
 // Check proximity to portal
 float distPortal = length(playerCam.pos - DISCO_PORTAL_POS);
 if (distPortal < 2.5f && sceneState == CITY_SCENE) {
  enterDiscoScene();  // Teleport + state change
 }
 if (distPortal < 2.5f && sceneState == DISCO_SCENE) {
  returnToCityScene();  // Return to city
 }
}

// In key callback (GLFW_KEY_1): enterDiscoScene()
// In key callback (GLFW_KEY_2): closeDialogue() only

void enterDiscoScene() {
 sceneState = SceneState::DISCO_SCENE;
 gameState = GameState::STATE_EXPLORING;
 playerCam.mode = CameraMode::FPS;
 playerCam.pos = DISCO_PORTAL_POS + glm::vec3(0, EYE_HEIGHT_DISCO, 0);
 playerCam.yaw = -90.0f;  // Face -Z into the room
 playerCam.pitch = 0.0f;
 // Rebuild disco scene geometry, reset disco uniforms
}

void returnToCityScene() {
 sceneState = SceneState::CITY_SCENE;
 gameState = GameState::STATE_EXPLORING;
 playerCam.mode = CameraMode::FPS;
 playerCam.pos = npcWorldPos + glm::vec3(3.0f, EYE_HEIGHT_CITY, 0.0f);
 playerCam.yaw = -90.0f; playerCam.pitch = 0.0f;
 // Rebuild city scene geometry
}
```

#### Dialogue UI — Console Menu Overlay

The dialogue UI is rendered as a full-screen textured quad overlay that draws the `dialogue_box.png` texture in the lower portion of the screen:

```cpp
1040:1075:discolib/src/renderer.cpp
#version 330 core
in vec2 vUV; out vec4 FragColor;
uniform sampler2D uDialogueTex;
uniform float uAlpha;
void main() {
 vec2 uv = vUV;
 // Map screen UV y=[0..0.38] to texture UV y=[0..1]
 uv.y = vUV.y / 0.38;
 vec4 col = texture(uDialogueTex, uv);
 col.a *= uAlpha;  // Fade-in/out support
 FragColor = col;
}
```

The UV remapping `vUV.y / 0.38` ensures the dialogue box texture (designed for the lower 38% of the screen) covers the correct area regardless of aspect ratio.

#### Neon Portal Ring Effect

```cpp
347:359:discolib/src/App.cpp
// NeonRingMesh: UV sphere ring at portal position
// NEON_VS / NEON_FS: ring glow shader
//   Ring mask: smoothstep(0.7, 0.9, dist) * smoothstep(1.0, 0.9, dist)
//   Glow: exp(-dist * 4.0) warm orange
//   Pulse: sin(time * 2.0) modulates glow intensity
```

### 3.5 `glDisable(GL_DEPTH_TEST)` for 2D Overlay Rendering

The rendering pipeline uses `glDisable(GL_DEPTH_TEST)` and `glDepthMask(GL_FALSE)` in three distinct contexts to ensure 2D overlay elements are rendered correctly:

#### 1. Post-Processing Screen Quad

```cpp
// App.cpp ~line 924
glBindFramebuffer(GL_FRAMEBUFFER, 0);           // Back to default framebuffer
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
glDisable(GL_DEPTH_TEST);                       // Disable depth testing
postProcessQuad.draw();                         // Full-screen quad with fisheye shader
```

Disabling depth testing ensures the screen-space post-process quad always renders on top of the 3D scene, regardless of its (arbitrary) depth value in NDC space.

#### 2. Dialogue UI Overlay

```cpp
1130:1134:discolib/src/renderer.cpp
// In Renderer::drawDialogueUI()
glEnable(GL_BLEND);
glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
glDepthMask(GL_FALSE);  // Disable depth writes
glDisable(GL_CULL_FACE);
// Draw dialogue quad (full screen, lower 38%)
dialogueQuad.draw();
glDepthMask(GL_TRUE);   // Re-enable depth writes
```

`glDepthMask(GL_FALSE)` prevents the dialogue quad from writing to the depth buffer, so the dialogue always appears on top of the 3D scene without interfering with depth testing for subsequent frames.

#### 3. Volumetric Light Cones and God Rays

```cpp
// App.cpp / renderer.cpp — volumetric beam rendering
glEnable(GL_BLEND);
glBlendFunc(GL_SRC_ALPHA, GL_ONE);  // Additive blending
glDepthMask(GL_FALSE);              // Volumetric cones don't write depth
glDisable(GL_CULL_FACE);            // Render both faces of cone meshes
// Draw all VOL_BEAM and GODRAY instances
glDepthMask(GL_TRUE);
glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);  // Restore alpha blending
```

Additive blending (`GL_SRC_ALPHA, GL_ONE`) combined with `glDepthMask(GL_FALSE)` creates the characteristic glow of volumetric light shafts — each layer adds to the accumulated brightness without obscuring geometry behind it.

---

## Summary of File References

| File | Role in Project |
|------|----------------|
| `discolib/include/discolib/types.hpp` | `Vertex` and `Mesh` data structures |
| `discolib/include/discolib/loader.hpp` | Loader declarations, `ShadowMap`, `DiscoFacet`, `GodRaySystem` |
| `discolib/include/discolib/model.hpp` | `Model` class, `AABB`, `MaterialData`, `MeshData` |
| `discolib/include/discolib/camera.hpp` | `Camera` struct, `CameraController`, `CameraMode` enum |
| `discolib/src/loader.cpp` | `loadOBJ`, `loadSTL`, `loadSTLWithPlanarUV`, `loadTexture`, `createShadowMap`, `createConeMesh`, `createGodRaySystem` |
| `discolib/src/model.cpp` | Assimp `Model::load`, `processMesh`, `loadMaterials` |
| `discolib/src/camera.cpp` | `outsideHexRoom`, `insideFurniture`, `isBlocked`, `applyCityCollision`, `Camera::viewMatrix`, `processInput` |
| `discolib/src/renderer.cpp` | All GLSL shaders (`SCENE_VS/FS`, `DEPTH_VS/FS`, `VOL_BEAM_VS/FS`, `GODRAY_VS/FS`, `CITY_VS/FS`, `SKYBOX_VS/FS`, `NEON_VS/FS`, `PP_VS/FS`, `DLG_VS/FS`), `calcShadow`, `buildPostProcessShaders` |
| `discolib/src/App.cpp` | `main` loop, state machine, `mouseCallback`, portal transitions, `enterDiscoScene`, `returnToCityScene`, `lightSpaceMatrix`, camera mode switching |
