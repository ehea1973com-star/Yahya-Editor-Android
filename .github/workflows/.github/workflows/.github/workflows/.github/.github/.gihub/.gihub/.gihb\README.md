
name: Build Yahya Editor APK

on:
  workflow_dispatch:
  push:

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: "8.9"

      - name: Generate Android project
        shell: bash
        run: |
          mkdir -p app/src/main/java/com/yahya/appeditor
          mkdir -p app/src/main

          cat > settings.gradle.kts <<'EOF'
          pluginManagement {
              repositories {
                  google()
                  mavenCentral()
                  gradlePluginPortal()
              }
          }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
              repositories {
                  google()
                  mavenCentral()
              }
          }
          rootProject.name = "YahyaEditor"
          include(":app")
          EOF

          cat > build.gradle.kts <<'EOF'
          plugins {
              id("com.android.application") version "8.7.3" apply false
              id("org.jetbrains.kotlin.android") version "2.0.21" apply false
              id("org.jetbrains.kotlin.plugin.compose") version "2.0.21" apply false
          }
          EOF

          cat > app/build.gradle.kts <<'EOF'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
              id("org.jetbrains.kotlin.plugin.compose")
          }

          android {
              namespace = "com.yahya.appeditor"
              compileSdk = 35

              defaultConfig {
                  applicationId = "com.yahya.appeditor"
                  minSdk = 26
                  targetSdk = 35
                  versionCode = 1
                  versionName = "1.0"
              }

              buildFeatures {
                  compose = true
              }
          }

          dependencies {
              implementation(platform("androidx.compose:compose-bom:2024.12.01"))
              implementation("androidx.activity:activity-compose:1.9.3")
              implementation("androidx.compose.ui:ui")
              implementation("androidx.compose.ui:ui-tooling-preview")
              implementation("androidx.compose.material3:material3")
          }
          EOF

          cat > app/src/main/AndroidManifest.xml <<'EOF'
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <application
                  android:label="يحيى لتعديل التطبيقات"
                  android:theme="@android:style/Theme.Material.NoActionBar"
                  android:supportsRtl="true"
                  android:allowBackup="true">
                  <activity
                      android:name=".MainActivity"
                      android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN"/>
                          <category android:name="android.intent.category.LAUNCHER"/>
                      </intent-filter>
                  </activity>
              </application>
          </manifest>
          EOF

          cat > app/src/main/java/com/yahya/appeditor/MainActivity.kt <<'EOF'
          package com.yahya.appeditor

          import android.content.Intent
          import android.net.Uri
          import android.os.Bundle
          import androidx.activity.ComponentActivity
          import androidx.activity.compose.BackHandler
          import androidx.activity.compose.setContent
          import androidx.activity.compose.rememberLauncherForActivityResult
          import androidx.activity.result.contract.ActivityResultContracts
          import androidx.compose.foundation.background
          import androidx.compose.foundation.layout.*
          import androidx.compose.foundation.rememberScrollState
          import androidx.compose.foundation.verticalScroll
          import androidx.compose.foundation.shape.RoundedCornerShape
          import androidx.compose.material3.*
          import androidx.compose.runtime.*
          import androidx.compose.ui.Alignment
          import androidx.compose.ui.Modifier
          import androidx.compose.ui.graphics.Color
          import androidx.compose.ui.platform.LocalContext
          import androidx.compose.ui.unit.LayoutDirection
          import androidx.compose.ui.platform.LocalLayoutDirection
          import androidx.compose.ui.text.font.FontWeight
          import androidx.compose.ui.unit.dp
          import androidx.compose.ui.unit.sp
          import androidx.compose.runtime.CompositionLocalProvider

          private val Gold = Color(0xFFD6B66C)
          private val Black = Color(0xFF101010)
          private val Panel = Color(0xFF202020)

          class MainActivity : ComponentActivity() {
              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  setContent { YahyaApp() }
              }
          }

          @Composable
          fun YahyaApp() {
              val context = LocalContext.current
              val prefs = remember {
                  context.getSharedPreferences("yahya", 0)
              }
              var page by remember { mutableStateOf("home") }
              var translation by remember {
                  mutableStateOf(prefs.getString("translation", "") ?: "")
              }
              var chosenFile by remember { mutableStateOf("لم يتم اختيار ملف") }
              var online by remember { mutableStateOf(false) }
              var message by remember { mutableStateOf("") }

              val apkPicker = rememberLauncherForActivityResult(
                  ActivityResultContracts.OpenDocument()
              ) { uri: Uri? ->
                  if (uri != null) {
                      chosenFile = uri.lastPathSegment ?: uri.toString()
                      message = "تم اختيار الملف. لم يتم تعديله بعد."
                  }
              }

              BackHandler(enabled = page != "home") {
                  page = "home"
                  message = ""
              }

              MaterialTheme(
                  colorScheme = darkColorScheme(
                      primary = Gold,
                      background = Black,
                      surface = Panel,
                      onPrimary = Black,
                      onBackground = Color.White,
                      onSurface = Color.White
                  )
              ) {
                  CompositionLocalProvider(
                      LocalLayoutDirection provides LayoutDirection.Rtl
                  ) {
                      Scaffold(
                          containerColor = Black,
                          topBar = {
                              Column(
                                  Modifier.fillMaxWidth()
                                      .background(Panel).padding(16.dp)
                              ) {
                                  Text(
                                      "يحيى",
                                      color = Gold,
                                      fontSize = 30.sp,
                                      fontWeight = FontWeight.Black
                                  )
                                  Text(
                                      "لتعديل التطبيقات",
                                      color = Color.White,
                                      fontWeight = FontWeight.Bold
                                  )
                              }
                          },
                          bottomBar = {
                              Row(
                                  Modifier.fillMaxWidth()
                                      .background(Panel)
                                      .navigationBarsPadding()
                                      .padding(4.dp),
                                  horizontalArrangement =
                                      Arrangement.SpaceEvenly
                              ) {
                                  listOf(
                                      "الرئيسية" to "home",
                                      "التعريب" to "editor",
                                      "APK" to "advanced",
                                      "روابطي" to "links"
                                  ).forEach { (label, destination) ->
                                      TextButton(
                                          onClick = { page = destination }
                                      ) {
                                          Text(
                                              label,
                                              color = if (page == destination)
                                                  Gold else Color.White,
                                              fontWeight = FontWeight.Bold
                                          )
                                      }
                                  }
                              }
                          }
                      ) { padding ->
                          Column(
                              Modifier.fillMaxSize()
                                  .padding(padding)
                                  .verticalScroll(rememberScrollState())
                                  .padding(16.dp),
                              verticalArrangement = Arrangement.spacedBy(14.dp)
                          ) {
                              when (page) {
                                  "home" -> {
                                      Text(
                                          "أهلاً بك في تطبيق يحيى",
                                          color = Gold,
                                          fontSize = 23.sp,
                                          fontWeight = FontWeight.Black
                                      )
                                      Text(
                                          "أدوات تعريب التطبيقات وإدارة الترجمات.",
                                          color = Color.White
                                      )
                                      ActionCard("التعريب اليدوي",
                                          "تحرير النصوص وحفظ الترجمة") {
                                          page = "editor"
                                      }
                                      ActionCard("إدارة ملفات APK",
                                          "اختر ملف تطبيق من هاتفك") {
                                          page = "advanced"
                                      }
                                      ActionCard("روابطي",
                                          "تلغرام ويوتيوب وفيسبوك") {
                                          page = "links"
                                      }
                                      Row(verticalAlignment = Alignment.CenterVertically) {
                                          Switch(
                                              checked = online,
                                              onCheckedChange = { online = it }
                                          )
                                          Spacer(Modifier.width(8.dp))
                                          Text(
                                              if (online) "الوضع المتصل"
                                              else "الوضع دون إنترنت",
                                              color = Gold
                                          )
                                      }
                                      Text(
                                          "التبديل يحدد الوضع داخل الواجهة فقط؛ " +
                                          "ولا يضيف خدمة ترجمة عبر الإنترنت.",
                                          color = Color.LightGray,
                                          fontSize = 12.sp
                                      )
                                  }

                                  "editor" -> {
                                      Text(
                                          "محرر الترجمة",
                                          color = Gold,
                                          fontSize = 22.sp,
                                          fontWeight = FontWeight.Black
                                      )
                                      OutlinedTextField(
                                          value = translation,
                                          onValueChange = {
                                              translation = it
                                          },
                                          modifier = Modifier.fillMaxWidth()
                                              .heightIn(min = 180.dp),
                                          label = { Text("اكتب النص أو الترجمة") },
                                          placeholder = {
                                              Text("اكتب الترجمة هنا...")
                                          }
                                      )
                                      Button(
                                          onClick = {
                                              prefs.edit()
                                                  .putString("translation", translation)
                                                  .apply()
                                              message = "تم حفظ النص على الجهاز."
                                          },
                                          modifier = Modifier.fillMaxWidth()
                                      ) {
                                          Text("حفظ الترجمة")
                                      }
                                      Text(
                                          "هذا محرر نصوص محلي؛ لا يعدّل موارد APK تلقائياً.",
                                          color = Color.LightGray
                                      )
                                  }

                                  "advanced" -> {
                                      Text(
                                          "إدارة APK",
                                          color = Gold,
                                          fontSize = 22.sp,
                                          fontWeight = FontWeight.Black
                                      )
                                      Button(
                                          onClick = {
                                              apkPicker.launch(
                                                  arrayOf(
                                                      "application/vnd.android.package-archive",
                                                      "application/octet-stream"
                                                  )
                                              )
                                          },
                                          modifier = Modifier.fillMaxWidth()
                                      ) {
                                          Text("اختيار ملف APK")
                                      }
                                      Text(chosenFile, color = Color.White)
                                      Text(
                                          "اختيار الملف لا يفكّه ولا يعيد بناءه. " +
                                          "أدوات تعديل الموارد والتوقيع تحتاج إلى دمج منفصل.",
                                          color = Color.LightGray
                                      )
                                  }

                                  "links" -> {
                                      Text(
                                          "روابطي",
                                          color = Gold,
                                          fontSize = 22.sp,
                                          fontWeight = FontWeight.Black
                                      )
                                      LinkButton("تلغرام",
                                          "https://t.me/ehea1973hmza") {
                                          openLink(context, it)
                                      }
                                      LinkButton("يوتيوب",
                                          "https://youtube.com/channel/UCmChpVHGV74adiyqncSyocg?si=9XQ_fkqIzcQNvxIM") {
                                          openLink(context, it)
                                      }
                                      LinkButton("فيسبوك",
                                          "https://www.facebook.com/share/g/1HGSjVh3c5/") {
                                          openLink(context, it)
                                      }
                                  }
                              }

                              if (message.isNotBlank()) {
                                  Text(message, color = Gold)
                              }
                          }
                      }
                  }
              }
          }

          @Composable
          fun ActionCard(
              title: String,
              description: String,
              onClick: () -> Unit
          ) {
              Card(
                  onClick = onClick,
                  modifier = Modifier.fillMaxWidth(),
                  shape = RoundedCornerShape(16.dp),
                  colors = CardDefaults.cardColors(containerColor = Panel)
              ) {
                  Column(Modifier.padding(16.dp)) {
                      Text(title, color = Gold, fontWeight = FontWeight.Black,
                          fontSize = 18.sp)
                      Spacer(Modifier.height(5.dp))
                      Text(description, color = Color.White)
                  }
              }
          }

          @Composable
          fun LinkButton(
              label: String,
              url: String,
              onOpen: (String) -> Unit
          ) {
              Button(
                  onClick = { onOpen(url) },
                  modifier = Modifier.fillMaxWidth()
              ) {
                  Text("فتح $label", fontWeight = FontWeight.Bold)
              }
          }

          private fun openLink(
              context: android.content.Context,
              url: String
          ) {
              runCatching {
                  context.startActivity(
                      Intent(Intent.ACTION_VIEW, Uri.parse(url))
                  )
              }
          }
          EOF

      - name: Build debug APK
        run: gradle --no-daemon assembleDebug

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: Yahya-Editor-APK
          path: app/build/outputs/apk/debug/app-debug.apk
          if-no-files-found: error
          retention-days: 7
