# .github-workflows-build.yml

on:
  push:
  workflow_dispatch:
permissions:
  contents: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Criar projeto
        run: |
          mkdir -p BrawlAI && cd BrawlAI
          mkdir -p app app/src/main app/src/main/java/com/brawlai/app app/src/main/res/values app/src/main/res/xml
          cat > settings.gradle.kts <<'BRAWLAI_EOF_7731'
          pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
              repositories { google(); mavenCentral() }
          }
          rootProject.name = "BrawlAI"
          include(":app")
          BRAWLAI_EOF_7731
          cat > build.gradle.kts <<'BRAWLAI_EOF_7731'
          plugins {
              id("com.android.application") version "8.7.3" apply false
              id("org.jetbrains.kotlin.android") version "2.0.21" apply false
          }
          BRAWLAI_EOF_7731
          cat > gradle.properties <<'BRAWLAI_EOF_7731'
          org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
          android.useAndroidX=true
          kotlin.code.style=official
          BRAWLAI_EOF_7731
          cat > app/build.gradle.kts <<'BRAWLAI_EOF_7731'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }
          android {
              namespace = "com.brawlai.app"
              compileSdk = 35
              defaultConfig {
                  applicationId = "com.brawlai.app"
                  minSdk = 30
                  targetSdk = 34
                  versionCode = 2
                  versionName = "2.0"
              }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions { jvmTarget = "17" }
          }
          BRAWLAI_EOF_7731
          cat > app/src/main/AndroidManifest.xml <<'BRAWLAI_EOF_7731'
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <queries>
                  <package android:name="com.supercell.brawlstars" />
              </queries>
              <application
                  android:allowBackup="false"
                  android:label="Brawl AI"
                  android:theme="@android:style/Theme.Material.NoActionBar"
                  android:supportsRtl="true">
                  <activity android:name=".MainActivity" android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>
                  <service
                      android:name=".BotService"
                      android:exported="true"
                      android:label="Brawl AI"
                      android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
                      <intent-filter>
                          <action android:name="android.accessibilityservice.AccessibilityService" />
                      </intent-filter>
                      <meta-data
                          android:name="android.accessibilityservice"
                          android:resource="@xml/accessibility_config" />
                  </service>
              </application>
          </manifest>
          BRAWLAI_EOF_7731
          cat > app/src/main/res/xml/accessibility_config.xml <<'BRAWLAI_EOF_7731'
          <?xml version="1.0" encoding="utf-8"?>
          <accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
              android:accessibilityEventTypes="typeWindowStateChanged"
              android:accessibilityFeedbackType="feedbackGeneric"
              android:canPerformGestures="true"
              android:canRetrieveWindowContent="true"
              android:canTakeScreenshot="true"
              android:notificationTimeout="100"
              android:description="@string/service_desc" />
          BRAWLAI_EOF_7731
          cat > app/src/main/res/values/strings.xml <<'BRAWLAI_EOF_7731'
          <resources>
              <string name="app_name">Brawl AI</string>
              <string name="service_desc">Quando o Brawl Stars está aberto, o Brawl AI vê a tela e faz toques/gestos para jogar sozinho. Botão STOP flutuante para parar.</string>
          </resources>
          BRAWLAI_EOF_7731
          cat > app/src/main/java/com/brawlai/app/Vision.kt <<'BRAWLAI_EOF_7731'
          package com.brawlai.app

          import android.graphics.Bitmap
          import android.graphics.Color

          class Blob(val x: Int, val y: Int, val w: Int, val h: Int, val count: Int) {
              val cx get() = x + w / 2f
              val cy get() = y + h / 2f
          }

          class Analysis(
              val w: Int, val h: Int,
              val inMatch: Boolean,
              val redRatio: Float,
              val yellowRatio: Float,
              val meX: Float, val meY: Float, val myBarW: Float,   // myBarW = 0 se nao achou
              val enemies: List<FloatArray>,                       // [x, y, larguraBarra]
              val button: FloatArray?,                             // [fx, fy] (fracao da tela) do botao amarelo grande
              val greenButton: FloatArray?,
              val sig: IntArray
          )

          /** Visao por cor: barras de vida vermelhas (inimigos), barra verde (eu), botao de ataque vermelho (partida), botao amarelo (menus). */
          class Vision {

              fun analyze(bmp: Bitmap): Analysis {
                  val w = bmp.width
                  val h = bmp.height
                  val n = w * h
                  val px = IntArray(n)
                  bmp.getPixels(px, 0, w, 0, 0, w, h)
                  val red = BooleanArray(n)
                  val green = BooleanArray(n)
                  val yellow = BooleanArray(n)
                  val hsv = FloatArray(3)
                  for (i in 0 until n) {
                      Color.colorToHSV(px[i], hsv)
                      val hh = hsv[0]
                      val s = hsv[1]
                      val v = hsv[2]
                      if (s > 0.55f && v > 0.55f) {
                          if (hh < 12f || hh > 348f) red[i] = true
                          else if (hh in 80f..165f) green[i] = true
                      }
                      if (s > 0.60f && v > 0.80f && hh in 36f..68f) yellow[i] = true
                  }
                  // assinatura de luminancia 16x9 (detecta tela parada = personagem preso)
                  val sig = IntArray(16 * 9)
                  for (gy in 0 until 9) for (gx in 0 until 16) {
                      val c = px[((gy * h) / 9 + h / 18) * w + (gx * w) / 16 + w / 32]
                      sig[gy * 16 + gx] = (Color.red(c) * 3 + Color.green(c) * 6 + Color.blue(c)) / 10
                  }

                  val redAtk = ratio(red, w, h, 0.74f, 0.58f, 0.99f, 0.97f)
                  val yelAtk = ratio(yellow, w, h, 0.74f, 0.58f, 0.99f, 0.97f)

                  // barras de vida (copias das mascaras, sem HUD)
                  val rb = red.copyOf()
                  val gb = green.copyOf()
                  for (m in arrayOf(rb, gb)) {
                      clear(m, w, h, 0f, 0f, 1f, 0.10f)      // HUD de cima
                      clear(m, w, h, 0.72f, 0.55f, 1f, 1f)   // botoes de ataque
                      clear(m, w, h, 0f, 0.60f, 0.30f, 1f)   // joystick
                      closeH(m, w, h, 3)
                  }
                  val off = 0.07f * h
                  val enemies = ArrayList<FloatArray>()
                  for (b in blobs(rb, w, h)) if (isBar(b, w, h)) enemies.add(floatArrayOf(b.cx, b.cy + off, b.w.toFloat()))
                  var meX = w / 2f
                  var meY = h / 2f
                  var myW = 0f
                  var bestD = Float.MAX_VALUE
                  for (b in blobs(gb, w, h)) if (isBar(b, w, h)) {
                      val d = Math.hypot((b.cx - w / 2f).toDouble(), (b.cy - h / 2f).toDouble()).toFloat()
                      if (d < 0.35f * h && d < bestD) { bestD = d; meX = b.cx; meY = b.cy + off; myW = b.w.toFloat() }
                  }
                  val meNearCenter = myW > 0f && bestD < 0.22f * h
                  val inMatch = yelAtk < 0.10f && (redAtk > 0.045f || meNearCenter)

                  var button: FloatArray? = null
                  var gbutton: FloatArray? = null
                  if (!inMatch) {
                      gbutton = bestButton(green, w, h, 0.006f)
                      button = bestButton(yellow, w, h, 0.004f)
                  }
                  return Analysis(w, h, inMatch, redAtk, yelAtk, meX, meY, myW, enemies, button, gbutton, sig)
              }

              private fun bestButton(mask: BooleanArray, w: Int, h: Int, minFrac: Float): FloatArray? {
                  val m = mask.copyOf()
                  closeH(m, w, h, 8)
                  var best: Blob? = null
                  for (b in blobs(m, w, h)) {
                      if (b.count < minFrac * w * h) continue
                      if (b.cy < 0.40f * h) continue
                      if (best == null || b.count > best.count) best = b
                  }
                  return best?.let { floatArrayOf(it.cx / w, it.cy / h) }
              }

              private fun isBar(b: Blob, w: Int, h: Int): Boolean =
                  b.w >= 0.03f * w && b.w <= 0.16f * w && b.h >= 0.006f * h && b.h <= 0.05f * h && b.w >= 2.5f * b.h

              private fun ratio(m: BooleanArray, w: Int, h: Int, x0: Float, y0: Float, x1: Float, y1: Float): Float {
                  val ax = (x0 * w).toInt(); val bx = (x1 * w).toInt()
                  val ay = (y0 * h).toInt(); val by = (y1 * h).toInt()
                  var c = 0
                  for (y in ay until by) for (x in ax until bx) if (m[y * w + x]) c++
                  return c.toFloat() / maxOf(1, (bx - ax) * (by - ay))
              }

              private fun clear(m: BooleanArray, w: Int, h: Int, x0: Float, y0: Float, x1: Float, y1: Float) {
                  for (y in (y0 * h).toInt() until (y1 * h).toInt()) for (x in (x0 * w).toInt() until (x1 * w).toInt()) m[y * w + x] = false
              }

              private fun closeH(m: BooleanArray, w: Int, h: Int, gap: Int) {
                  for (y in 0 until h) {
                      var last = -1000
                      for (x in 0 until w) {
                          if (m[y * w + x]) {
                              val g = x - last - 1
                              if (g in 1..gap) for (k in last + 1 until x) m[y * w + k] = true
                              last = x
                          }
                      }
                  }
              }

              private fun blobs(mask: BooleanArray, w: Int, h: Int): List<Blob> {
                  val n = w * h
                  val seen = BooleanArray(n)
                  val stack = IntArray(n)
                  val out = ArrayList<Blob>()
                  for (s in 0 until n) {
                      if (!mask[s] || seen[s]) continue
                      var sp = 0
                      stack[sp++] = s
                      seen[s] = true
                      var x0 = w; var y0 = h; var x1 = 0; var y1 = 0; var cnt = 0
                      while (sp > 0) {
                          val i = stack[--sp]
                          val x = i % w
                          val y = i / w
                          cnt++
                          if (x < x0) x0 = x
                          if (x > x1) x1 = x
                          if (y < y0) y0 = y
                          if (y > y1) y1 = y
                          if (x > 0 && mask[i - 1] && !seen[i - 1]) { seen[i - 1] = true; stack[sp++] = i - 1 }
                          if (x < w - 1 && mask[i + 1] && !seen[i + 1]) { seen[i + 1] = true; stack[sp++] = i + 1 }
                          if (y > 0 && mask[i - w] && !seen[i - w]) { seen[i - w] = true; stack[sp++] = i - w }
                          if (y < h - 1 && mask[i + w] && !seen[i + w]) { seen[i + w] = true; stack[sp++] = i + w }
                      }
                      out.add(Blob(x0, y0, x1 - x0 + 1, y1 - y0 + 1, cnt))
                  }
                  return out
              }
          }
          BRAWLAI_EOF_7731
          cat > app/src/main/java/com/brawlai/app/Brain.kt <<'BRAWLAI_EOF_7731'
          package com.brawlai.app

          import android.content.Context
          import kotlin.math.hypot
          import kotlin.random.Random

          /** Parametros de comportamento que a IA evolui. */
          class Params(
              var prefDist: Float = 0.30f,   // distancia preferida do inimigo (fracao da altura)
              var band: Float = 0.07f,
              var strafe: Float = 0.70f,     // segundos entre trocas de lado (zigue-zague)
              var fireRange: Float = 0.55f,
              var superRange: Float = 0.42f,
              var retreatHp: Float = 0.30f,
              var dodge: Float = 0.50f
          ) {
              fun toArray() = floatArrayOf(prefDist, band, strafe, fireRange, superRange, retreatHp, dodge)
              companion object {
                  val LO = floatArrayOf(0.15f, 0.03f, 0.30f, 0.35f, 0.25f, 0.10f, 0.20f)
                  val HI = floatArrayOf(0.50f, 0.15f, 1.60f, 0.80f, 0.65f, 0.60f, 1.00f)
                  fun from(a: FloatArray) = Params(a[0], a[1], a[2], a[3], a[4], a[5], a[6])
              }
          }

          /** "Evolucao": hill-climbing. Cada partida testa uma variacao do melhor comportamento; se pontuar melhor, vira o novo melhor. */
          class Brain(ctx: Context) {
              private val sp = ctx.getSharedPreferences("brain", Context.MODE_PRIVATE)
              var matches = sp.getInt("matches", 0)
                  private set
              var bestScore = sp.getFloat("best", 0f)
                  private set
              private var best: FloatArray = (sp.getString("params", null)?.split(",")?.mapNotNull { it.toFloatOrNull() }?.toFloatArray()
                  ?.takeIf { it.size == 7 }) ?: Params().toArray()

              fun sample(): Params {
                  val sigma = 0.30f / (1f + matches / 40f) + 0.05f
                  val a = FloatArray(7) { i ->
                      val range = Params.HI[i] - Params.LO[i]
                      (best[i] + (Random.nextFloat() * 2f - 1f) * sigma * range).coerceIn(Params.LO[i], Params.HI[i])
                  }
                  return Params.from(a)
              }

              fun report(p: Params, reward: Float) {
                  matches++
                  if (matches <= 3 || reward >= bestScore * 0.97f) {
                      best = p.toArray()
                      bestScore = if (matches <= 3) maxOf(bestScore, reward) else 0.7f * bestScore + 0.3f * reward
                  } else {
                      bestScore *= 0.98f   // deixa o "melhor" envelhecer para a IA continuar se adaptando
                  }
                  sp.edit().putInt("matches", matches).putFloat("best", bestScore)
                      .putString("params", best.joinToString(",")).apply()
              }
          }

          class Move(val dx: Float, val dy: Float, val fire: Boolean, val sup: Boolean, val engaged: Boolean)

          class Policy(private val p: Params, private val H: Float) {
              private var sign = if (Random.nextBoolean()) 1 else -1
              private var flipAt = 0.0
              private var dodgeUntil = 0.0
              private var dodgeSign = 1
              private var wander = rand()
              private var wanderUntil = 0.0
              private var unstickUntil = 0.0
              private var unstick = floatArrayOf(0f, 0f)

              private fun rand(): FloatArray {
                  val a = Random.nextDouble(0.0, 2 * Math.PI)
                  return floatArrayOf(Math.cos(a).toFloat(), Math.sin(a).toFloat())
              }

              fun decide(now: Double, meX: Float, meY: Float, enemies: List<FloatArray>, hp: Float?, dmg: Boolean, stuck: Boolean): Move {
                  if (stuck && now > unstickUntil) { unstickUntil = now + 0.9; unstick = rand() }
                  if (now < unstickUntil) return Move(unstick[0], unstick[1], false, false, false)
                  if (enemies.isEmpty()) {
                      if (now >= wanderUntil) { wander = rand(); wanderUntil = now + Random.nextDouble(1.2, 3.0) }
                      return Move(wander[0], wander[1], false, false, false)
                  }
                  val t = enemies.minByOrNull { (it[0] - meX) * (it[0] - meX) + (it[1] - meY) * (it[1] - meY) }!!
                  val vx = t[0] - meX
                  val vy = t[1] - meY
                  val dist = hypot(vx, vy).coerceAtLeast(1f)
                  val ux = vx / dist
                  val uy = vy / dist
                  val px = -uy
                  val py = ux
                  val d = dist / H
                  if (now >= flipAt) { sign = -sign; flipAt = now + p.strafe * Random.nextDouble(0.7, 1.3) }
                  if (dmg && now > dodgeUntil) { dodgeUntil = now + p.dodge; dodgeSign = if (Random.nextBoolean()) 1 else -1 }
                  val dodging = now < dodgeUntil
                  var mx = 0f
                  var my = 0f
                  if (hp != null && hp < p.retreatHp) { mx = -ux; my = -uy }
                  else if (d > p.prefDist + p.band) { mx = ux; my = uy }
                  else if (d < p.prefDist - p.band) { mx = -ux; my = -uy }
                  val s = if (dodging) dodgeSign else sign
                  val wgt = if (dodging) 1.2f else 0.7f
                  mx += px * s * wgt
                  my += py * s * wgt
                  val n = hypot(mx, my).coerceAtLeast(0.001f)
                  return Move(mx / n, my / n, d <= p.fireRange, d <= p.superRange, true)
              }
          }
          BRAWLAI_EOF_7731
          cat > app/src/main/java/com/brawlai/app/BotService.kt <<'BRAWLAI_EOF_7731'
          package com.brawlai.app

          import android.accessibilityservice.AccessibilityService
          import android.accessibilityservice.AccessibilityService.GestureResultCallback
          import android.accessibilityservice.AccessibilityService.ScreenshotResult
          import android.accessibilityservice.AccessibilityService.TakeScreenshotCallback
          import android.accessibilityservice.GestureDescription
          import android.content.ContentValues
          import android.content.Context
          import android.graphics.Bitmap
          import android.graphics.Color
          import android.graphics.Path
          import android.graphics.PixelFormat
          import android.graphics.Typeface
          import android.os.Handler
          import android.os.Looper
          import android.os.SystemClock
          import android.provider.MediaStore
          import android.view.Display
          import android.view.Gravity
          import android.view.MotionEvent
          import android.view.View
          import android.view.WindowManager
          import android.view.accessibility.AccessibilityEvent
          import android.widget.LinearLayout
          import android.widget.TextView
          import java.util.concurrent.Executors

          class BotService : AccessibilityService() {

              companion object {
                  const val GAME = "com.supercell.brawlstars"
                  const val SEG = 220L           // duracao de cada trecho do gesto de movimento (ms)
