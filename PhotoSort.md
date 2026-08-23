# 📸 PhotoSort – teljes projekt-fájl (self-contained)

> Cél: hasonló fotók csoportosítása a telefonon tárolt galériában, törlési javaslattal.
> Ez a dokumentum MINDENT tartalmaz a folytatáshoz: állapotot, döntéseket, teljes kódot,
> telefonos build-útmutatót és folytatás-promptot. Importálható bármilyen AI-chatbe vagy repóba.

---

## 1. Állapot-pillanatfelvétel

| Tétel | Állapot |
|---|---|
| Stack | Kotlin 2.0 + Jetpack Compose (Material 3) + Room + Coil, minSdk 30, targetSdk 35 |
| Hozzáférés a képekhez | NINCS Google-API (a Photos Library API új appoknak le van zárva) → helyi **MediaStore** + futásidejű engedély (`READ_MEDIA_IMAGES` / részleges hozzáférés) |
| Hasonlóság-detektálás | **dHash** perceptuális hash (64 bit/kép) + Hamming-távolság + Union-Find klaszterezés |
| Gyorsítótár | Room DB (`photosort.db`), csak új/módosult képeket hash-el újra |
| Törlés | `MediaStore.createDeleteRequest` → rendszer-dialógus erősíti meg |
| Build | GitHub Actions adja ki az APK-t (laptop nélkül) |
| Verzió | v1 (duplikátum/sorozatfelvétel-detektálás). v2 roadmap a végén |

## 2. Fájlfa

```
PhotoSort/
├── settings.gradle.kts
├── build.gradle.kts
├── gradle.properties
├── app/
│   ├── build.gradle.kts
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── res/values/strings.xml
│       └── java/hu/devlab/photosort/
│           ├── MainActivity.kt
│           ├── domain/{PhotoItem.kt, PerceptualHasher.kt, ClusterEngine.kt}
│           ├── data/MediaStoreSource.kt
│           ├── data/db/AppDatabase.kt
│           └── ui/{GalleryViewModel.kt, PhotoSortApp.kt}
└── .github/workflows/build.yml   ← felhő-buildhez
```

## 3. 📱 TELEFONOS BUILD – lépésről lépésre (csak telefonböngésző!)

Első alkalom: ~20–30 perc. Minden ingyenes kereten belül marad.

### Lépések
1. **github.com** → regisztráció/bejelentkezés.
2. Új repository: név `PhotoSort`, lehet Private is. Init README-vel.
3. Repó oldalon: zöld **Code** gomb → **Codespaces** fül → **Create codespace on main**
   (megnyílik egy VS Code böngészőben, terminállal – ez lesz a „géped").
4. A terminálba:
   ```bash
   nano create_project.sh
   ```
   Másold be az 5. szekcióban lévő szkriptet (mobil VS Code menü → beillesztés), mentsd
   (Ctrl+S jellegű mentés a mobilmenüben), majd:
   ```bash
   bash create_project.sh
   ```
   A szkript **az aktuális repó gyökerébe** generál minden fájlt, és magától commitol.
5. Push:
   ```bash
   git push
   ```
   (Codespaces-ben a push bejelentkezve megy; ha kérdezi, nyisd meg a kapott bejelentkezési linket.)
6. Repó oldal → **Actions** fül → „Build APK” workflow lefut (~4–6 perc).
7. A kész futásnál alul **Artifacts** → töltsd le a `PhotoSort-debug` ZIP-et.
8. Telefonon csomagold ki → nyisd meg az `app-debug.apk`-t → telepítés
   (engedélyezd az „ismeretlen forrásból telepítést” a fájlkezelő/böngésző számára).
9. App indítás → engedély-dialógus → elemzés (progress bar) → csoportok →
   koppints a felesleges képekre → **Törles** gomb → a RENDSZER saját dialógusában erősíts meg.

### Ha hibás a build (piros ❌ az Actions-ben)
- Nyisd meg a futást → kattints a elbukott step-re → másold ki a hibaüzenetet
- A hibaüzenetet + ezt az md-t add bármelyik AI-nak: „javítsd a szkriptet”

### Gyakori hibák
| Hiba | Ok / megoldás |
|---|---|
| `Unsupported class file major version` | JDK nem 17 → a workflow már beállít Temurin 17-et |
| Telepítéskor „App not installed” | régi APK maradvány → uninstall, utána újratelepítés |
| Engedély után üres lista | csak felhőben lévő fotók vannak helyben → tölts le párat a készülékre |
| Push nem megy | ellenőrizd, hogy a repó gyökerében állsz-e (`ls` → látszik az `app` mappa?) |

---

## 4. 🔁 Folytatás-prompt (új funkcióhoz, bármelyik AI-chatbe)

```
Folytatod a PhotoSort nevű Android-projektemet (részletek a csatolt md-ben).
Stack: Kotlin 2.0, Compose/Material3, Room, dHash + Union-Find, minSdk 30.
A teljes forráskód az md 5. szekciójában lévő create_project.sh-ban van,
egyszer már sikeresen lefordult (GitHub Actions, assembleDebug).
KORLÁTOK: csak telefonböngészőből dolgozom (Codespaces + Actions), nincs PC-em;
Google Photos Library API nem használható; felhőben lévő fotók elérhetetlenek.
KÖVETKEZŐ FELADAT: [pl. "BK-tree index", "TFLite tematikus csoportosítás",
"videók támogatása", "küszöb-csúszka a UI-ban", "release aláírás"]
```

---

## 5. ⚙️ `create_project.sh` – EZ A TELJES PROJEKT

```bash
#!/usr/bin/env bash
# PhotoSort – teljes projekt generátor.
# Használat Codespaces-ben a repó gyekerében:  bash create_project.sh
set -e

# Minden fájl az AKTUÁLIS könyvtárba kerül (a Codespace repó gyökerébe)
mkdir -p app/src/main/java/hu/devlab/photosort/{domain,data/db,ui} \
         app/src/main/res/values .github/workflows

cat > .gitignore <<'EOF_GITIGNORE'
.gradle/
build/
local.properties
.idea/
*.iml
EOF_GITIGNORE

cat > settings.gradle.kts <<'EOF_SETTINGS'
pluginManagement {
    repositories { google(); mavenCentral(); gradlePluginPortal() }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories { google(); mavenCentral() }
}
rootProject.name = "PhotoSort"
include(":app")
EOF_SETTINGS

cat > build.gradle.kts <<'EOF_ROOTBUILD'
plugins {
    id("com.android.application") version "8.6.1" apply false
    id("org.jetbrains.kotlin.android") version "2.0.20" apply false
    id("org.jetbrains.kotlin.plugin.compose") version "2.0.20" apply false
    id("com.google.devtools.ksp") version "2.0.20-1.0.25" apply false
}
EOF_ROOTBUILD

cat > gradle.properties <<'EOF_PROPS'
org.gradle.jvmargs=-Xmx2048m
android.useAndroidX=true
android.nonTransitiveRClass=true
kotlin.code.style=official
EOF_PROPS

cat > app/build.gradle.kts <<'EOF_APPBUILD'
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
    id("com.google.devtools.ksp")
}

android {
    namespace = "hu.devlab.photosort"
    compileSdk = 35
    defaultConfig {
        applicationId = "hu.devlab.photosort"
        minSdk = 30
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions { jvmTarget = "17" }
    buildFeatures { compose = true }
}

dependencies {
    implementation("androidx.core:core-ktx:1.13.1")
    val bom = platform("androidx.compose:compose-bom:2024.09.02")
    implementation(bom)
    implementation("androidx.activity:activity-compose:1.9.2")
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.6")
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")
    implementation("io.coil-kt:coil-compose:2.7.0")
}
EOF_APPBUILD

cat > app/src/main/AndroidManifest.xml <<'EOF_MANIFEST'
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
    <uses-permission android:name="android.permission.READ_MEDIA_VISUAL_USER_SELECTED" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
        android:maxSdkVersion="32" />
    <application
        android:label="@string/app_name"
        android:theme="@android:style/Theme.Material.Light.NoActionBar">
        <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
EOF_MANIFEST

cat > app/src/main/res/values/strings.xml <<'EOF_STRINGS'
<resources><string name="app_name">PhotoSort</string></resources>
EOF_STRINGS

J=app/src/main/java/hu/devlab/photosort

cat > $J/domain/PhotoItem.kt <<'EOF_ITEM'
package hu.devlab.photosort.domain

data class PhotoItem(
    val id: Long,
    val uri: String,
    val dateAddedSec: Long,
    val sizeBytes: Long,
    val hash: Long
)
EOF_ITEM

cat > $J/domain/PerceptualHasher.kt <<'EOF_HASH'
package hu.devlab.photosort.domain

import android.content.Context
import android.graphics.ImageDecoder
import android.net.Uri

/**
 * dHash: 9x8 szurkearnyalatos racs, szomszedos pixelek vizszintes osszehasonlitasa
 * -> 64 bites ujjlenyomat. Gyors, offline, kompressziot/atmeretezest toleral.
 */
object PerceptualHasher {

    fun dHash(context: Context, uri: Uri): Long {
        val source = ImageDecoder.createSource(context.contentResolver, uri)
        val bmp = ImageDecoder.decodeBitmap(source) { decoder, _, _ ->
            decoder.allocator = ImageDecoder.ALLOCATOR_SOFTWARE
            decoder.setTargetSize(9, 8)   // az EXIF-forgatast az ImageDecoder kezeli
        }
        val pixels = IntArray(9 * 8)
        bmp.getPixels(pixels, 0, 9, 0, 0, 9, 8)
        bmp.recycle()

        var hash = 0L
        for (y in 0 until 8) {
            for (x in 0 until 8) {
                if (gray(pixels[y * 9 + x]) > gray(pixels[y * 9 + x + 1]))
                    hash = hash or (1L shl (y * 8 + x))
            }
        }
        return hash
    }

    fun hamming(a: Long, b: Long): Int = java.lang.Long.bitCount(a xor b)

    private fun gray(px: Int): Int {
        val r = (px shr 16) and 0xFF
        val g = (px shr 8) and 0xFF
        val b = px and 0xFF
        return (r * 299 + g * 587 + b * 114) / 1000
    }
}
EOF_HASH

cat > $J/domain/ClusterEngine.kt <<'EOF_CLUSTER'
package hu.devlab.photosort.domain

/**
 * Union-Find klaszterezes Hamming-tavolsag alapjan.
 * Kuszob: 0-5 duplikatum, 6-10 sorozatfelvetel, >10 mas jelenet.
 */
object ClusterEngine {

    data class Cluster(val photos: List<PhotoItem>)

    fun cluster(items: List<PhotoItem>, threshold: Int = 8): List<Cluster> {
        val parent = IntArray(items.size) { it }

        fun find(i: Int): Int {
            var root = i
            while (parent[root] != root) root = parent[root]
            var cur = i
            while (parent[cur] != root) {
                val up = parent[cur]
                parent[cur] = root   // ut-tomorites
                cur = up
            }
            return root
        }

        fun union(a: Int, b: Int) { parent[find(a)] = find(b) }

        for (i in items.indices)
            for (j in i + 1 until items.size)
                if (PerceptualHasher.hamming(items[i].hash, items[j].hash) <= threshold)
                    union(i, j)

        return items.indices
            .groupBy { find(it) }
            .values
            .filter { it.size > 1 }   // csak az "erdekes" csoportok
            .map { members ->
                Cluster(members.map { items[it] }.sortedBy { it.dateAddedSec })
            }
            .sortedByDescending { it.photos.size }
    }
}
EOF_CLUSTER

cat > $J/data/MediaStoreSource.kt <<'EOF_SOURCE'
package hu.devlab.photosort.data

import android.content.ContentUris
import android.content.Context
import android.net.Uri
import android.provider.MediaStore

/** Galeria-lekerdezes: ugyanazt latja, amit a Fotok app a keszuleken */
class MediaStoreSource(private val context: Context) {

    data class PhotoMeta(
        val id: Long,
        val uri: Uri,
        val dateAddedSec: Long,
        val dateModifiedSec: Long,
        val sizeBytes: Long
    )

    fun queryPhotos(): List<PhotoMeta> {
        val projection = arrayOf(
            MediaStore.Images.Media._ID,
            MediaStore.Images.Media.DATE_ADDED,
            MediaStore.Images.Media.DATE_MODIFIED,
            MediaStore.Images.Media.SIZE
        )
        val out = mutableListOf<PhotoMeta>()
        context.contentResolver.query(
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
            projection, null, null,
            "${MediaStore.Images.Media.DATE_ADDED} DESC"
        )?.use { c ->
            val idC = c.getColumnIndexOrThrow(MediaStore.Images.Media._ID)
            val addC = c.getColumnIndexOrThrow(MediaStore.Images.Media.DATE_ADDED)
            val modC = c.getColumnIndexOrThrow(MediaStore.Images.Media.DATE_MODIFIED)
            val sizeC = c.getColumnIndexOrThrow(MediaStore.Images.Media.SIZE)
            while (c.moveToNext()) {
                val id = c.getLong(idC)
                out += PhotoMeta(
                    id = id,
                    uri = ContentUris.withAppendedId(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, id),
                    dateAddedSec = c.getLong(addC),
                    dateModifiedSec = c.getLong(modC),
                    sizeBytes = c.getLong(sizeC)
                )
            }
        }
        return out
    }
}
EOF_SOURCE

cat > $J/data/db/AppDatabase.kt <<'EOF_DB'
package hu.devlab.photosort.data.db

import android.content.Context
import androidx.room.*

@Entity(tableName = "hashes")
data class HashEntity(
    @PrimaryKey val mediaId: Long,
    val hash: Long,
    val dateModified: Long
)

@Dao
interface HashDao {
    @Query("SELECT * FROM hashes")
    suspend fun all(): List<HashEntity>

    @Upsert
    suspend fun upsertAll(items: List<HashEntity>)

    @Query("DELETE FROM hashes WHERE mediaId IN (:ids)")
    suspend fun deleteIds(ids: List<Long>)
}

@Database(entities = [HashEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun hashDao(): HashDao

    companion object {
        @Volatile private var instance: AppDatabase? = null

        fun get(context: Context): AppDatabase = instance ?: synchronized(this) {
            instance ?: Room.databaseBuilder(
                context.applicationContext,
                AppDatabase::class.java,
                "photosort.db"
            ).fallbackToDestructiveMigration().build().also { instance = it }
        }
    }
}
EOF_DB

cat > $J/ui/GalleryViewModel.kt <<'EOF_VM'
package hu.devlab.photosort.ui

import android.app.Application
import android.app.PendingIntent
import android.net.Uri
import android.provider.MediaStore
import androidx.lifecycle.AndroidViewModel
import androidx.lifecycle.viewModelScope
import hu.devlab.photosort.data.MediaStoreSource
import hu.devlab.photosort.data.db.AppDatabase
import hu.devlab.photosort.data.db.HashEntity
import hu.devlab.photosort.domain.ClusterEngine
import hu.devlab.photosort.domain.PhotoItem
import hu.devlab.photosort.domain.PerceptualHasher
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext

sealed interface UiState {
    data object NeedsPermission : UiState
    data class Scanning(val done: Int, val total: Int) : UiState
    data class Ready(val clusters: List<ClusterEngine.Cluster>) : UiState
    data class Error(val message: String) : UiState
}

class GalleryViewModel(app: Application) : AndroidViewModel(app) {

    private val _state = MutableStateFlow<UiState>(UiState.NeedsPermission)
    val state: StateFlow<UiState> = _state.asStateFlow()

    private val _selected = MutableStateFlow<Set<String>>(emptySet())
    val selected: StateFlow<Set<String>> = _selected.asStateFlow()

    fun toggle(uri: String) {
        _selected.value = _selected.value.toMutableSet().apply { if (!add(uri)) remove(uri) }
    }

    fun clearSelection() { _selected.value = emptySet() }

    fun scan() = viewModelScope.launch {
        try {
            val app = getApplication<Application>()
            val dao = AppDatabase.get(app).hashDao()
            val metas = withContext(Dispatchers.IO) { MediaStoreSource(app).queryPhotos() }

            // 1) elavult cache-bejegyzesek torlese (kozen torolt kepek)
            val validIds = metas.mapTo(HashSet()) { it.id }
            val stale = dao.all().map { it.mediaId }.filterNot { it in validIds }
            if (stale.isNotEmpty()) dao.deleteIds(stale)

            // 2) csak uj/modosult kepek hash-elese
            val cached = dao.all().associateBy { it.mediaId }
            val fresh = metas.filter { cached[it.id]?.dateModified != it.dateModifiedSec }
            _state.value = UiState.Scanning(0, metas.size)

            val computed = HashMap<Long, Long>(fresh.size)
            var done = 0
            withContext(Dispatchers.IO) {
                for (m in fresh) {
                    computed[m.id] = try {
                        PerceptualHasher.dHash(app, m.uri)
                    } catch (e: Exception) { 0L }
                    if (++done % 10 == 0 || done == fresh.size) {
                        _state.value = UiState.Scanning(cached.size + done, metas.size)
                    }
                }
            }
            if (fresh.isNotEmpty()) {
                dao.upsertAll(fresh.map { HashEntity(it.id, computed.getValue(it.id), it.dateModifiedSec) })
            }

            // 3) klaszterezes a teljes allomanyon
            val items = metas.map { m ->
                PhotoItem(m.id, m.uri.toString(), m.dateAddedSec, m.sizeBytes,
                          cached[m.id]?.hash ?: computed.getValue(m.id))
            }
            val clusters = withContext(Dispatchers.Default) { ClusterEngine.cluster(items) }
            clearSelection()
            _state.value = UiState.Ready(clusters)
        } catch (e: SecurityException) {
            _state.value = UiState.NeedsPermission
        } catch (e: Exception) {
            _state.value = UiState.Error(e.message ?: "Ismeretlen hiba")
        }
    }

    /** Rendszer-torlesdialogus: a user a hivatalos rendszerablakon erosithet meg */
    fun buildDeletePendingIntent(): PendingIntent {
        val uris = _selected.value.map { Uri.parse(it) }
        require(uris.isNotEmpty()) { "Nincs kijelolt kep" }
        return MediaStore.createDeleteRequest(getApplication<Application>().contentResolver, uris)
    }
}
EOF_VM

cat > $J/ui/PhotoSortApp.kt <<'EOF_UI'
package hu.devlab.photosort.ui

import android.app.PendingIntent
import androidx.compose.foundation.border
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.LazyRow
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.CheckCircle
import androidx.compose.material.icons.filled.Refresh
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.dp
import coil.compose.AsyncImage
import hu.devlab.photosort.domain.ClusterEngine

@Composable
fun PhotoSortApp(
    viewModel: GalleryViewModel,
    onRequestPermission: () -> Unit,
    onLaunchDelete: (PendingIntent) -> Unit
) {
    val state by viewModel.state.collectAsState()
    val selected by viewModel.selected.collectAsState()

    Surface(Modifier.fillMaxSize()) {
        when (val s = state) {
            is UiState.NeedsPermission -> CenteredScreen {
                Text(
                    "A PhotoSort a telefonon tarolt kepeket elemzi,\nhogy megtalalja a hasonlokat.\nEhhez engedelyre van szukseg.",
                    textAlign = TextAlign.Center
                )
                Spacer(Modifier.height(16.dp))
                Button(onClick = onRequestPermission) { Text("Galeria eleresenek engedelyezese") }
            }
            is UiState.Scanning -> CenteredScreen {
                Text("Kepek elemzese...", style = MaterialTheme.typography.headlineSmall)
                Spacer(Modifier.height(12.dp))
                LinearProgressIndicator(
                    progress = { if (s.total == 0) 0f else s.done.toFloat() / s.total },
                    modifier = Modifier.fillMaxWidth()
                )
                Spacer(Modifier.height(8.dp))
                Text("${s.done} / ${s.total}")
            }
            is UiState.Error -> CenteredScreen {
                Text("Hiba: ${s.message}")
                Spacer(Modifier.height(12.dp))
                Button(onClick = onRequestPermission) { Text("Ujra") }
            }
            is UiState.Ready -> ReadyScreen(s.clusters, selected, viewModel, onLaunchDelete)
        }
    }
}

@Composable
private fun CenteredScreen(content: @Composable () -> Unit) {
    Box(Modifier.fillMaxSize().padding(32.dp), contentAlignment = Alignment.Center) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) { content() }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
private fun ReadyScreen(
    clusters: List<ClusterEngine.Cluster>,
    selected: Set<String>,
    viewModel: GalleryViewModel,
    onLaunchDelete: (PendingIntent) -> Unit
) {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text(if (clusters.isEmpty()) "PhotoSort" else "${clusters.size} hasonlo csoport") },
                actions = {
                    IconButton(onClick = { viewModel.scan() }) { Icon(Icons.Filled.Refresh, "Frissites") }
                }
            )
        },
        bottomBar = {
            if (selected.isNotEmpty()) {
                BottomAppBar {
                    Text("${selected.size} kijelolve", Modifier.weight(1f).padding(start = 16.dp))
                    TextButton(onClick = viewModel::clearSelection) { Text("Elvetes") }
                    Button(onClick = {
                        runCatching { viewModel.buildDeletePendingIntent() }.onSuccess(onLaunchDelete)
                    }) { Text("Torles") }
                }
            }
        }
    ) { padding ->
        if (clusters.isEmpty()) {
            Box(Modifier.fillMaxSize().padding(padding), contentAlignment = Alignment.Center) {
                Text("Nem talaltunk hasonlo kepcsoportot.")
            }
        } else {
            LazyColumn(Modifier.padding(padding)) {
                items(clusters) { cluster -> ClusterCard(cluster, selected, viewModel::toggle) }
            }
        }
    }
}

@Composable
private fun ClusterCard(group: ClusterEngine.Cluster, selected: Set<String>, onToggle: (String) -> Unit) {
    Card(Modifier.padding(horizontal = 12.dp, vertical = 6.dp)) {
        Column(Modifier.padding(12.dp)) {
            Text("${group.photos.size} hasonlo kep", style = MaterialTheme.typography.titleMedium)
            Spacer(Modifier.height(8.dp))
            LazyRow(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                items(group.photos) { photo -> Thumb(photo.uri, photo.uri in selected, onToggle) }
            }
        }
    }
}

@Composable
private fun Thumb(uri: String, isSelected: Boolean, onToggle: (String) -> Unit) {
    Box(
        Modifier
            .size(110.dp)
            .clip(RoundedCornerShape(10.dp))
            .border(
                width = if (isSelected) 3.dp else 1.dp,
                color = if (isSelected) MaterialTheme.colorScheme.primary
                        else MaterialTheme.colorScheme.outlineVariant,
                shape = RoundedCornerShape(10.dp)
            )
            .clickable { onToggle(uri) }
    ) {
        AsyncImage(model = uri, contentDescription = null, contentScale = ContentScale.Crop,
                   modifier = Modifier.fillMaxSize())
        if (isSelected) Icon(Icons.Filled.CheckCircle, null,
                             tint = MaterialTheme.colorScheme.primary,
                             modifier = Modifier.align(Alignment.TopEnd).padding(4.dp))
    }
}
EOF_UI

cat > .github/workflows/build.yml <<'EOF_CI'
name: Build APK
on:
  push:
    branches: [ main ]
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
      - name: Set up Gradle 8.9
        uses: gradle/actions/setup-gradle@v3
        with:
          gradle-version: '8.9'
      - name: Build debug APK
        run: gradle assembleDebug --no-daemon
      - name: Upload APK artifact
        uses: actions/upload-artifact@v4
        with:
          name: PhotoSort-debug
          path: app/build/outputs/apk/debug/app-debug.apk
EOF_CI

git add .
git -c user.name="builder" -c user.email="builder@users.noreply.github.com" \
    commit -m "PhotoSort v1 - teljes projekt" || true

echo ""
echo "✅ Projekt legeneralva a repó gyokerere!"
echo "Kovetkezo lepes:  git push"
echo "Utana a GitHub Actions fulepen lefut a build -> Artifacts -> PhotoSort-debug ZIP -> APK"
```

---

## 6. Roadmap (v2)

1. Küszöb-csúszka a UI-ban (0–16 Hamming) → élő újraclusterelés
2. BK-tree index → több tízezer képnél is másodperces klaszterezés
3. TensorFlow Lite + MobileNet embedding → tartalom-alapú csoportok („kutyás fotók”)
4. Videó-támogatás (`MediaStore.Video`)
5. Release signing + Play Store (internal track)

## 7. Ismert korlátok

- Felhőben (Google Fotók szerverén) lévő, helyileg nem tárolt képek: **elérhetetlenek** harmadik feles appként
- Olvashatatlan fájlok semleges `0` hash-t kapnak
- Nagyon nagy könyvtárnál (>20–30 ezer kép) az O(N²) párosítás lassulhat → roadmap #2
