/*
 TAMIL VALARI — ARMY GRID ARRAY & BALLISTIC TRAJECTORIES
 Processing Java Mode / Android APDE — P3D

 FEATURES:
 - Enemies arranged in a perfect grid array (platoons and artillery lines).
 - Evasive riding through dynamic battlefield.
 - Soldiers & Cannons use ballistic math to PREDICT player movement and fire accurately.
 - Visible muzzle flashes, smoke puffs, and bullet trails for enemies.
 - Fully parametric Valari throws with 4 tweakable sliders.
 - Sliders adjust Flight Timing, Inclination, Lift, and Drag dynamically on the run.
 - Valari can strike enemies.
*/

import java.util.ArrayList;

// -------------------- STATES --------------------

final int MODE_DRAW = 0;
final int MODE_3D = 1;

final int H_READY = 0;
final int H_RUN = 1;
final int H_STOP = 2;

final int V_READY = 0;
final int V_FLY = 1;
final int V_RET = 2;

// -------------------- PARAMETERS --------------------

final float BASE_SPEED = 20.0;
final float RUN_SPEED = 22.0; 
final float GRAVITY = 9.81;
final float PPM = 14.0;

final int MAX_POINTS = 1600;
final int MAX_SCRIBBLE = 600;
final int MAX_SAVED = 12;

// -------------------- GLOBAL STATE --------------------

int viewMode = MODE_DRAW;
int horseState = H_READY;
int valariState = V_READY;

boolean isDead = false;
boolean autoThrow = true; 
boolean drawing = false;

float runnerX = 0, runnerY = 0;
float runnerHeading = 0; 
float runDist = 0, runCycle = 0;

float animTime = 0, spinPhase = 0;
float launchHeading = 0;
float cannonFireTimer = 0;

float camX = radians(28);
float camY = radians(-35);
float lastMX, lastMY;

int throwCount = 0;
int pathCol;
String status = "Draw an evasive trajectory, then tap RIDE.";

float btnY, btnH;
float rideX, rideW, stopX, stopW;
float throwX, throwW, drawX, drawW;
float viewX, viewW, autoX, autoW;
float resetX, resetW;

// -------------------- DATA CLASSES --------------------

class V3 {
  float x, y, z;
  V3(float X, float Y, float Z) { x = X; y = Y; z = Z; }
}

class FlightPoint {
  float x, y, z, t, speed;
  FlightPoint(float X, float Y, float Z, float T, float S) {
    x = X; y = Y; z = Z; t = T; speed = S;
  }
}

class Soldier {
  float x, y, heading, fireTimer, flashTimer;
  Soldier(float x, float y, float h) {
    this.x = x; this.y = y; this.heading = h;
    this.fireTimer = random(1.0, 4.0);
    this.flashTimer = 0;
  }
}

class CannonPos {
  float x, y, heading, flashTimer;
  CannonPos(float x, float y, float h) {
    this.x = x; this.y = y; this.heading = h;
    this.flashTimer = 0;
  }
}

class MusketBall {
  float x, y, z, vx, vy, vz;
  boolean active = true;
  ArrayList<V3> trail = new ArrayList<V3>(); 

  MusketBall(float X, float Y, float Z, float VX, float VY, float VZ) {
    x = X; y = Y; z = Z; vx = VX; vy = VY; vz = VZ;
  }
  
  void update(float dt) {
    if (!active) {
      if (trail.size() > 0) trail.remove(0); 
      return; 
    }
    trail.add(new V3(x, y, z));
    if (trail.size() > 12) trail.remove(0); 
    x += vx * dt; y += vy * dt; z += vz * dt;
    vz -= GRAVITY * dt; 
    if (z <= 0.0) { z = 0; active = false; }
  }
  
  void draw3D() {
    if (trail.size() > 1) {
      noFill(); stroke(255, 220, 150, 180); strokeWeight(1.5); 
      beginShape();
      for (V3 p : trail) vertex(p.x * PPM, -p.z * PPM, p.y * PPM);
      vertex(x * PPM, -z * PPM, y * PPM);
      endShape();
    }
    if (!active) return;
    pushMatrix(); translate(x * PPM, -z * PPM, y * PPM);
    fill(255, 255, 200); noStroke(); sphereDetail(3); sphere(1.5);
    popMatrix();
  }
}

class Cannonball {
  float x, y, z, vx, vy, vz;
  boolean active = true;
  ArrayList<V3> trail = new ArrayList<V3>(); 

  Cannonball(float X, float Y, float Z, float VX, float VY, float VZ) {
    x = X; y = Y; z = Z; vx = VX; vy = VY; vz = VZ;
  }
  
  void update(float dt) {
    if (!active) {
      if (trail.size() > 0) trail.remove(0); 
      return; 
    }
    trail.add(new V3(x, y, z));
    if (trail.size() > 25) trail.remove(0);
    x += vx * dt; y += vy * dt; z += vz * dt;
    vz -= GRAVITY * dt; 
    if (z <= 0.0) { z = 0; active = false; }
  }
  
  void draw3D() {
    if (trail.size() > 1) {
      noFill(); stroke(255, 80, 0, 200); strokeWeight(3);
      beginShape();
      for (V3 p : trail) vertex(p.x * PPM, -p.z * PPM, p.y * PPM);
      vertex(x * PPM, -z * PPM, y * PPM);
      endShape();
    }
    if (!active) return;
    pushMatrix(); translate(x * PPM, -z * PPM, y * PPM);
    fill(30); noStroke(); sphereDetail(6); sphere(3.5);
    popMatrix();
  }
}

class Particle {
  float x, y, z, vx, vy, vz;
  int col; float size, alpha;
  float rotX, rotY, rotZ, vrX, vrY, vrZ;
  int type; 

  Particle(float X, float Y, float Z, int c, float s, int t) {
    x = X; y = Y; z = Z; col = c; size = s; type = t; alpha = 255;
    if (type == 2) {
      vx = random(-5, 5); vy = random(-5, 5); vz = random(1, 4);
      vrX = random(-2, 2); vrY = random(-2, 2); vrZ = random(-2, 2);
    } else {
      vx = random(-15, 15); vy = random(-15, 15); vz = random(4, 22);
      vrX = random(-8, 8); vrY = random(-8, 8); vrZ = random(-8, 8);
    }
    rotX = random(TWO_PI); rotY = random(TWO_PI); rotZ = random(TWO_PI);
  }
  
  void update(float dt) {
    if (type == 2) { 
      size += dt * 4.0; alpha -= dt * 300; 
    } else { 
      alpha -= dt * 60; 
    }
    if (alpha <= 0) return;

    if (type == 1 && z <= 0.06) return; 
    
    x += vx * dt; y += vy * dt; z += vz * dt;
    if (type != 2) { 
      vz -= GRAVITY * dt;
      if (z < 0.05) {
        if (type == 1) {
          z = 0.05 + random(0, 0.02); vx = 0; vy = 0; vz = 0; vrX = 0; vrY = 0; vrZ = 0;
        } else {
          z = 0.05; vz = -vz * 0.4; 
          vx *= 0.7; vy *= 0.7; vrX *= 0.5; vrY *= 0.5; vrZ *= 0.5;
        }
      }
    }
    rotX += vrX * dt; rotY += vrY * dt; rotZ += vrZ * dt;
  }
  
  void draw3D() {
    if (alpha <= 0) return;
    pushMatrix(); translate(x * PPM, -z * PPM, y * PPM);
    if (z <= 0.1 && type == 1) {
      rotateX(HALF_PI); fill(col, alpha); noStroke();
      ellipse(0, 0, size * PPM * 4, size * PPM * 4); 
    } else {
      rotateX(rotX); rotateY(rotY); rotateZ(rotZ);
      fill(col, alpha); noStroke();
      if (type == 0) box(size * PPM);
      else { sphereDetail(type == 2 ? 3 : 4); sphere(size * PPM); }
    }
    popMatrix();
  }
}

class ThrowModel {
  int id, col;
  float inclination, CL, CD, alpha_v; 
  float releaseX, releaseY, releaseZ, initialSpeed, runSpeed;
  float duration, maxHeight, range, randomAlteration; 

  ThrowModel(int I, int C, float INC, float CL_val, float CD_val, float ALPHA,
             float X, float Y, float Z, float V, float RV, float flightTime) {
    id = I; col = C; inclination = INC; CL = CL_val; CD = CD_val; alpha_v = ALPHA;
    releaseX = X; releaseY = Y; releaseZ = Z; runSpeed = RV;
    randomAlteration = (RV > 0) ? random(-RV, RV) * 0.5 : 0;
    initialSpeed = V + randomAlteration;
    duration = flightTime; 
    
    range = max(10.0, 20.0 + (CL * 15.0) - (CD * 30.0) + randomAlteration); 
    maxHeight = 2.0 + (CL * 3.0);
  }
}

class SavedFlight {
  ArrayList<FlightPoint> points = new ArrayList<FlightPoint>();
  int col;
  SavedFlight(ArrayList<FlightPoint> src, int C) {
    col = C;
    for (FlightPoint p : src) points.add(new FlightPoint(p.x, p.y, p.z, p.t, p.speed));
  }
}

class UISlider {
  float x, y, w, h, minVal, maxVal, val;
  String label;
  UISlider(String l, float x, float y, float w, float h, float minV, float maxV, float start) {
    this.label = l; this.x = x; this.y = y; this.w = w; this.h = h;
    this.minVal = minV; this.maxVal = maxV; this.val = start;
  }
  void draw() {
    noStroke(); fill(45, 55, 70); rect(x, y, w, h, 6);
    fill(255); textAlign(CENTER, CENTER); textSize(max(9, height*0.014));
    text(label + ": " + nf(val, 1, 2), x + w/2, y + h*0.25);
    float cy = y + h*0.7;
    stroke(200); strokeWeight(3); line(x + 10, cy, x + w - 10, cy);
    float k = map(val, minVal, maxVal, x + 10, x + w - 10);
    noStroke(); fill(255, 215, 40); ellipse(k, cy, 14, 14);
  }
  boolean hit(float mx, float my) { return mx >= x && mx <= x+w && my >= y && my <= y+h; }
  void drag(float mx) {
    float u = constrain((mx - (x + 10)) / max(1.0, w - 20), 0, 1);
    val = lerp(minVal, maxVal, u);
  }
}

ArrayList<V3> scribble = new ArrayList<V3>();
ArrayList<FlightPoint> flight = new ArrayList<FlightPoint>();
ArrayList<SavedFlight> saved = new ArrayList<SavedFlight>();
ArrayList<Soldier> randomSoldiers = new ArrayList<Soldier>(); 
ArrayList<CannonPos> cannons = new ArrayList<CannonPos>();
ArrayList<Cannonball> cannonballs = new ArrayList<Cannonball>();
ArrayList<MusketBall> musketballs = new ArrayList<MusketBall>();
ArrayList<Particle> particles = new ArrayList<Particle>();
ArrayList<UISlider> sliders = new ArrayList<UISlider>();

UISlider activeSlider = null;
ThrowModel model = null;
V3 pos = new V3(0, 0, 4.6);
V3 vel = new V3(0, 0, 0);

// ====================================================
// INITIALIZATION
// ====================================================

void settings() { 
  size(1280, 720, P3D); 
  smooth(4); 
}

void setup() {
  orientation(LANDSCAPE);
  frameRate(60);
  layoutUI();
  pathCol = color(230, 55, 195);
  spawnEnemies();
}

void spawnEnemies() {
  randomSoldiers.clear();
  cannons.clear();
  
  // PERFECT ARRAY: Cannons arranged in artillery lines
  for (float cx = -450; cx <= 450; cx += 150) {
    for (float cy = -450; cy <= 450; cy += 150) {
      if (abs(cx) < 120 && abs(cy) < 120) continue; // Safe inner zone
      cannons.add(new CannonPos(cx, cy, 0));
    }
  }
  
  // PERFECT ARRAY: Soldiers arranged in strict grid platoons
  for (float sx = -550; sx <= 550; sx += 60) {
    for (float sy = -550; sy <= 550; sy += 60) {
      if (abs(sx) < 90 && abs(sy) < 90) continue; // Safe inner zone
      // Tiny random offset to make them look like individuals standing in formation
      randomSoldiers.add(new Soldier(sx + random(-4, 4), sy + random(-4, 4), 0));
    }
  }
}

void layoutUI() {
  btnY = height * 0.025; btnH = height * 0.09;
  float bw = width * 0.11; float gap = width * 0.015;
  float cx = gap;
  
  rideX = cx; rideW = bw; cx += bw + gap;
  stopX = cx; stopW = bw; cx += bw + gap;
  throwX = cx; throwW = bw; cx += bw + gap;
  drawX = cx; drawW = bw; cx += bw + gap;
  viewX = cx; viewW = bw; cx += bw + gap;
  autoX = cx; autoW = bw; cx += bw + gap;
  resetX = cx; resetW = bw; 

  sliders.clear();
  float px = width*0.73, py = height*0.48, pw = width*0.25, ph = height*0.1;
  sliders.add(new UISlider("Flight Timing (s)", px, py, pw, ph, 1.5, 10.0, 4.8)); py += ph*1.15;
  sliders.add(new UISlider("Inclination (deg)", px, py, pw, ph, 5.0, 85.0, 75.0)); py += ph*1.15;
  sliders.add(new UISlider("Lift Coeff (C_L)", px, py, pw, ph, 0.1, 3.0, 1.5)); py += ph*1.15;
  sliders.add(new UISlider("Drag Coeff (C_D)", px, py, pw, ph, 0.01, 0.5, 0.12)); 
}

// ====================================================
// MAIN LOOP & PHYSICS
// ====================================================

void draw() {
  float dt = constrain(1.0 / max(frameRate, 1), 0, 0.05);
  updateFrame(dt);
  if (viewMode == MODE_DRAW) drawOrthographic();
  else drawScene3D();
  drawHUD();
}

void updateFrame(float dt) {
  if (horseState == H_RUN && !isDead) {
    runCycle += dt * 10;
    if (scribble.size() > 1) {
      runDist += RUN_SPEED * dt;
      updateRunnerPosition();
    } else {
      runnerX += RUN_SPEED * dt;
      runnerHeading = 0; 
    }
    if (random(1) < 0.6) {
      particles.add(new Particle(runnerX - cos(runnerHeading)*1.5, runnerY - sin(runnerHeading)*1.5, 
                                 0, color(180, 160, 130), random(0.5, 1.5), 2));
    }
  }

  if (valariState == V_FLY) {
    animTime += dt;
    integrateFlight(dt);
    checkValariEnemyHits();
  }

  updateCannonballs(dt);
  updateMusketballs(dt);
  
  for (int i = particles.size() - 1; i >= 0; i--) {
    Particle p = particles.get(i);
    p.update(dt);
    if (p.alpha <= 0) particles.remove(i); 
  }

  checkCannonballHits();
  checkMusketballHits();

  if (!isDead && (horseState == H_RUN || valariState == V_FLY)) {
    updateEnemiesFiring(dt);
  }
}

void updateCannonballs(float dt) {
  for (int i = cannonballs.size() - 1; i >= 0; i--) {
    Cannonball cb = cannonballs.get(i); 
    cb.update(dt);
    if (!cb.active && cb.trail.size() == 0) cannonballs.remove(i);
  }
}

void updateMusketballs(float dt) {
  for (int i = musketballs.size() - 1; i >= 0; i--) {
    MusketBall mb = musketballs.get(i); 
    mb.update(dt);
    if (!mb.active && mb.trail.size() == 0) musketballs.remove(i);
  }
}

void checkCannonballHits() {
  if (isDead || horseState == H_READY) return;
  for (Cannonball c : cannonballs) {
    if (c.active && distance2D(c.x, c.y, runnerX, runnerY) < 2.2 && c.z < 4.5) {
       triggerShatter("ARTILLERY");
       c.active = false;
       break;
    }
  }
}

void checkMusketballHits() {
  if (isDead || horseState == H_READY) return;
  for (MusketBall m : musketballs) {
    if (m.active && distance2D(m.x, m.y, runnerX, runnerY) < 1.2 && m.z < 2.5) {
       triggerShatter("MUSKET");
       m.active = false;
       break;
    }
  }
}

void updateEnemiesFiring(float dt) {
  cannonFireTimer += dt;
  
  for (CannonPos cp : cannons) if (cp.flashTimer > 0) cp.flashTimer -= dt;
  
  if (cannonFireTimer >= 0.25 && cannons.size() > 0) { 
    cannonFireTimer = 0;
    ArrayList<CannonPos> activeThreats = new ArrayList<CannonPos>();
    for (CannonPos cp : cannons) {
       // Huge range so array can fire
       if (distance2D(runnerX, runnerY, cp.x, cp.y) < 350.0) activeThreats.add(cp);
    }
    if (activeThreats.size() > 0) {
      CannonPos cp = activeThreats.get((int)random(activeThreats.size()));
      fireCannonball(cp);
      cp.flashTimer = 0.15; 
    }
  }
  
  for (Soldier s : randomSoldiers) {
    if (s.flashTimer > 0) s.flashTimer -= dt;
    if (distance2D(s.x, s.y, runnerX, runnerY) < 200.0) { // Large sight range
      s.heading = atan2(runnerY - s.y, runnerX - s.x); 
      s.fireTimer -= dt;
      if (s.fireTimer <= 0) {
        fireMusket(s);
        s.fireTimer = random(1.5, 4.0); 
        s.flashTimer = 0.1; 
      }
    }
  }
}

void fireCannonball(CannonPos cp) {
  float dist = distance2D(cp.x, cp.y, runnerX, runnerY);
  float flightTime = max(0.2, dist / 40.0); // Avg bullet speed 40
  
  // BALLISTIC PREDICTION: Lead the target mathematically
  float pX = runnerX + (horseState == H_RUN ? cos(runnerHeading) * RUN_SPEED * flightTime : 0);
  float pY = runnerY + (horseState == H_RUN ? sin(runnerHeading) * RUN_SPEED * flightTime : 0);
  
  float vx = (pX - cp.x) / flightTime;
  float vy = (pY - cp.y) / flightTime;
  // Calculate arc so z=0 exact when it reaches player
  float vz = (0.0 - 1.2 + 0.5 * GRAVITY * flightTime * flightTime) / flightTime;
  
  cp.heading = atan2(vy, vx); 
  cannonballs.add(new Cannonball(cp.x, cp.y, 1.2, vx, vy, vz));
  
  for (int i=0; i<5; i++) {
     particles.add(new Particle(cp.x + cos(cp.heading)*2, cp.y + sin(cp.heading)*2, 1.5, color(150, 150, 150), random(2.0, 4.0), 2));
  }
}

void fireMusket(Soldier s) {
  float dist = distance2D(s.x, s.y, runnerX, runnerY);
  float flightTime = max(0.1, dist / 95.0); // Fast bullet speed 95
  
  // BALLISTIC PREDICTION: Lead the target mathematically
  float pX = runnerX + (horseState == H_RUN ? cos(runnerHeading) * RUN_SPEED * flightTime : 0);
  float pY = runnerY + (horseState == H_RUN ? sin(runnerHeading) * RUN_SPEED * flightTime : 0);
  
  float vx = (pX - s.x) / flightTime;
  float vy = (pY - s.y) / flightTime;
  float vz = (0.0 - 1.0 + 0.5 * GRAVITY * flightTime * flightTime) / flightTime;
  
  s.heading = atan2(vy, vx);
  musketballs.add(new MusketBall(s.x, s.y, 1.0, vx, vy, vz));
  
  particles.add(new Particle(s.x + cos(s.heading)*1.5, s.y + sin(s.heading)*1.5, 1.5, color(200, 200, 200), 1.5, 2));
}

void triggerShatter(String cause) {
  isDead = true; horseState = H_STOP; 
  status = "FATAL HIT: INTERCEPTED BY " + cause + " FIRE!";
  for (int i=0; i<45; i++) particles.add(new Particle(runnerX, runnerY, random(0.5, 3.0), color(100, 50, 20), random(0.3, 0.8), 0));
  for (int i=0; i<30; i++) particles.add(new Particle(runnerX, runnerY, random(2.5, 4.5), color(150, 85, 50), random(0.2, 0.5), 0));
  for (int i=0; i<250; i++) particles.add(new Particle(runnerX, runnerY, random(1.0, 4.5), color(200, 10, 10), random(0.1, 0.25), 1));
}

void checkValariEnemyHits() {
  for (int i = randomSoldiers.size() - 1; i >= 0; i--) {
    Soldier s = randomSoldiers.get(i);
    if (distance2D(s.x, s.y, pos.x, pos.y) < 3.5 && abs(pos.z - 1.5) < 3.0) {
      for(int k=0; k<25; k++) particles.add(new Particle(s.x, s.y, 1.5, color(200, 10, 10), random(0.2, 0.5), 1));
      randomSoldiers.remove(i);
    }
  }
}

void updateRunnerPosition() {
  if (scribble.size() < 2) return;
  float currentDist = 0;
  for (int i = 0; i < scribble.size() - 1; i++) {
    V3 p1 = scribble.get(i), p2 = scribble.get(i + 1);
    float segLen = distance2D(p1.x, p1.y, p2.x, p2.y);
    if (currentDist + segLen >= runDist) {
      float t = (runDist - currentDist) / segLen;
      runnerX = lerp(p1.x, p2.x, t); runnerY = lerp(p1.y, p2.y, t);
      runnerHeading = atan2(p2.y - p1.y, p2.x - p1.x);
      return;
    }
    currentDist += segLen;
  }
  V3 lastP = scribble.get(scribble.size() - 1);
  runnerX = lastP.x; runnerY = lastP.y;
  horseState = H_STOP; status = "Horse reached the end of the evasive maneuver.";
}

float distance2D(float x1, float y1, float x2, float y2) { return sqrt(sq(x1 - x2) + sq(y1 - y2)); }
float speedOf(V3 v) { return sqrt(v.x * v.x + v.y * v.y + v.z * v.z); }

// ====================================================
// 3D PARAMETRIC RETURN TRAJECTORY 
// ====================================================

void integrateFlight(float dt) {
  if (model == null || valariState != V_FLY) return;
  float progress = min(1.0, animTime / model.duration);
  
  float theta = progress * TWO_PI;
  float R = model.range / 2.0; 
  float H = model.maxHeight; 

  float bx = R * sin(theta);
  float by = R * (1 - cos(theta));
  float bz = H * sin(theta / 2.0);

  float tiltRad = radians(90 - model.inclination);
  float tiltY = by * cos(tiltRad) - bz * sin(tiltRad);
  float tiltZ = by * sin(tiltRad) + bz * cos(tiltRad);

  float yaw = launchHeading;
  float tx = bx * cos(yaw) - tiltY * sin(yaw);
  float ty = bx * sin(yaw) + tiltY * cos(yaw);
  float tz = tiltZ;

  pos.x = model.releaseX + tx;
  pos.y = model.releaseY + ty;
  pos.z = model.releaseZ + tz;

  float w = TWO_PI / model.duration;
  float dbx = R * w * cos(theta);
  float dby = R * w * sin(theta);
  float dbz = H * (w / 2.0) * cos(theta / 2.0);

  float dtiltY = dby * cos(tiltRad) - dbz * sin(tiltRad);
  float dtiltZ = dby * sin(tiltRad) + dbz * cos(tiltRad);

  vel.x = dbx * cos(yaw) - dtiltY * sin(yaw);
  vel.y = dbx * sin(yaw) + dtiltY * cos(yaw);
  vel.z = dtiltZ;

  spinPhase += (20.0 + model.CL * 10.0) * dt; 

  if (flight.size() < MAX_POINTS && animTime <= model.duration) {
    flight.add(new FlightPoint(pos.x, pos.y, pos.z, animTime, speedOf(vel)));
  }
  if (progress >= 1.0) finishFlight();
}

// ====================================================
// 2D DRAW VIEW
// ====================================================

float centerX() { return width * 0.43; }
float centerY() { return height * 0.57; }
float worldToScreenX(float x) { return centerX() + (x - runnerX) * PPM; }
float worldToScreenY(float y) { return centerY() - (y - runnerY) * PPM; }
V3 screenToWorld(float sx, float sy) { return new V3(runnerX + (sx - centerX()) / PPM, runnerY + (centerY() - sy) / PPM, 0); }

void drawOrthographic() {
  background(220, 230, 233);
  noStroke(); fill(245); rect(0, height*0.13, width, height*0.87);

  stroke(190, 202, 208); strokeWeight(1);
  float step = 5 * PPM;
  for (float x = centerX() % step; x < width; x += step) line(x, height*0.14, x, height);
  for (float y = height*0.14; y < height; y += step) line(0, y, width, y);

  for (SavedFlight s : saved) {
    noFill(); stroke(s.col, 85); strokeWeight(2);
    beginShape();
    for (FlightPoint p : s.points) vertex(worldToScreenX(p.x), worldToScreenY(p.y));
    endShape();
  }

  if (scribble.size() > 1) {
    noFill(); stroke(pathCol); strokeWeight(4);
    beginShape();
    for (V3 p : scribble) vertex(worldToScreenX(p.x), worldToScreenY(p.y));
    endShape();
  }

  fill(80); noStroke();
  for (Soldier s : randomSoldiers) {
    ellipse(worldToScreenX(s.x), worldToScreenY(s.y), 4, 4);
    if (s.flashTimer > 0) { 
      fill(255, 200, 50); 
      ellipse(worldToScreenX(s.x + cos(s.heading)*1.5), worldToScreenY(s.y + sin(s.heading)*1.5), 6, 6);
      fill(80);
    }
  }
  
  fill(40); noStroke();
  for (CannonPos cp : cannons) {
    ellipse(worldToScreenX(cp.x), worldToScreenY(cp.y), 7, 7);
    if (cp.flashTimer > 0) { 
      fill(255, 150, 30); 
      ellipse(worldToScreenX(cp.x + cos(cp.heading)*2.5), worldToScreenY(cp.y + sin(cp.heading)*2.5), 10, 10);
      fill(40);
    }
  }

  for (Cannonball cb : cannonballs) { fill(255, 60, 40); ellipse(worldToScreenX(cb.x), worldToScreenY(cb.y), 5, 5); }
  for (MusketBall mb : musketballs) { fill(200); ellipse(worldToScreenX(mb.x), worldToScreenY(mb.y), 3, 3); }

  if (!isDead) {
    noStroke(); fill(100, 50, 20);
    ellipse(worldToScreenX(runnerX), worldToScreenY(runnerY), 22, 22);
    fill(255, 220, 60); ellipse(worldToScreenX(runnerX), worldToScreenY(runnerY), 8, 8);
  }
}

// ====================================================
// 3D VIEW
// ====================================================

void drawScene3D() {
  background(150, 202, 230);
  pushMatrix();
  translate(width*0.38, height*0.78, -70);
  rotateX(camX); rotateY(camY);

  float focusX = runnerX, focusY = runnerY;
  if (model != null && (valariState == V_FLY || valariState == V_RET)) {
    focusX = model.releaseX; focusY = model.releaseY;
  }
  translate(-focusX*PPM, 0, -focusY*PPM);

  lights(); ambientLight(125, 125, 125);
  directionalLight(255, 255, 245, -0.5, 1, -0.5);

  drawGround(); 
  drawRandomSoldiers(); 
  for (CannonPos cp : cannons) drawCannon(cp);
  for (Cannonball cb : cannonballs) cb.draw3D();
  for (MusketBall mb : musketballs) mb.draw3D();
  
  for (SavedFlight s : saved) {
    noFill(); stroke(s.col, 90); strokeWeight(2);
    beginShape();
    for (FlightPoint p : s.points) vertex(p.x*PPM, -p.z*PPM, p.y*PPM);
    endShape();
  }
  if (scribble.size() > 1) {
    noFill(); stroke(pathCol, 190); strokeWeight(2);
    beginShape();
    for (V3 p : scribble) vertex(p.x*PPM, -0.03*PPM, p.y*PPM);
    endShape();
  }
  if (flight.size() >= 2) {
    noFill(); stroke(model != null ? model.col : pathCol); strokeWeight(4);
    beginShape();
    for (FlightPoint p : flight) vertex(p.x*PPM, -p.z*PPM, p.y*PPM);
    endShape();
  }

  for (Particle p : particles) p.draw3D();
  
  if (!isDead) drawRiderOnHorse();
  if (valariState == V_FLY) drawFlyingValari();
  else if (!isDead) drawHeldValari();

  popMatrix();
}

void drawGround() {
  pushMatrix(); rotateX(PI/2); rectMode(CENTER);
  noStroke(); fill(110, 165, 85); rect(0, 0, 12000, 12000);
  stroke(255, 255, 255, 55); strokeWeight(1);
  for (float i = -6000; i <= 6000; i += 5*PPM) { line(i, -6000, i, 6000); line(-6000, i, 6000, i); }
  popMatrix();
}

void drawRandomSoldiers() {
  for (Soldier s : randomSoldiers) {
    if (distance2D(s.x, s.y, runnerX, runnerY) > 350) continue;
    pushMatrix();
    translate(s.x * PPM, 0, s.y * PPM); rotateY(-s.heading + HALF_PI);
    fill(240);
    pushMatrix(); translate(-3, -10, 0); box(4, 20, 4); popMatrix();
    pushMatrix(); translate(3, -10, 0); box(4, 20, 4); popMatrix();
    fill(200, 30, 30);
    pushMatrix(); translate(0, -28, 0); box(12, 16, 8); popMatrix();
    pushMatrix(); translate(-8, -28, 0); box(4, 16, 4); popMatrix();
    pushMatrix(); translate(8, -28, 0); box(4, 16, 4); popMatrix();
    fill(255, 200, 150);
    pushMatrix(); translate(0, -40, 0); box(6, 6, 6); popMatrix();
    fill(20);
    pushMatrix(); translate(5, -30, 6); rotateX(radians(-75)); box(2, 28, 2); 
    if (s.flashTimer > 0) { 
      translate(0, -15, 0); 
      fill(255, 200, 50, 200); noStroke();
      sphere(6);
    }
    popMatrix();
    popMatrix();
  }
}

void drawCannon(CannonPos cp) {
  if (distance2D(cp.x, cp.y, runnerX, runnerY) > 450) return;
  pushMatrix(); translate(cp.x * PPM, 0, cp.y * PPM); rotateY(-cp.heading + HALF_PI); 
  fill(100, 60, 30); pushMatrix(); translate(0, -10, 0); box(16, 8, 20); popMatrix();
  fill(40); pushMatrix(); translate(-10, -10, 0); box(4, 16, 16); popMatrix();
  pushMatrix(); translate(10, -10, 0); box(4, 16, 16); popMatrix();
  fill(20); pushMatrix(); translate(0, -16, 4); rotateX(radians(20)); box(6, 6, 24); 
  if (cp.flashTimer > 0) { 
     translate(0, 0, 13);
     fill(255, 150, 30, 220); noStroke();
     sphere(10);
  }
  popMatrix();
  popMatrix();
}

void drawRiderOnHorse() {
  pushMatrix();
  translate(runnerX * PPM, 0, runnerY * PPM); rotateY(-runnerHeading); 
  float hBob = horseState == H_RUN ? abs(sin(runCycle * 2.0)) * 6 : 0; 
  translate(0, -hBob, 0);

  fill(100, 50, 20); 
  pushMatrix(); translate(0, -35, 0); box(40, 20, 20); popMatrix();
  pushMatrix(); translate(15, -50, 0); rotateZ(radians(30)); box(12, 25, 10); popMatrix();
  
  float hSwing = horseState == H_RUN ? sin(runCycle * 2.0) * 0.7 : 0; 
  fill(60, 30, 15);
  pushMatrix(); translate(12, -20, -7); rotateZ(hSwing); translate(0, 10, 0); box(6, 22, 6); popMatrix();
  pushMatrix(); translate(12, -20, 7); rotateZ(-hSwing); translate(0, 10, 0); box(6, 22, 6); popMatrix();
  pushMatrix(); translate(-12, -20, -7); rotateZ(-hSwing); translate(0, 10, 0); box(6, 22, 6); popMatrix();
  pushMatrix(); translate(-12, -20, 7); rotateZ(hSwing); translate(0, 10, 0); box(6, 22, 6); popMatrix();

  translate(-4, -50, 0); 
  fill(150, 85, 50); pushMatrix(); translate(0, -10, 0); box(12, 20, 14); popMatrix();
  fill(70, 42, 28); pushMatrix(); translate(0, -26, 0); sphereDetail(10); sphere(7); popMatrix();
  
  fill(45, 45, 55);
  pushMatrix(); translate(0, -15, 8); rotateZ(valariState == V_FLY ? radians(25) : radians(-15)); translate(0, 9, 0); box(5, 18, 6); popMatrix();
  popMatrix();
}

void drawBoomerang() {
  pushMatrix(); scale(0.9); noStroke();
  fill(135, 65, 28);
  pushMatrix(); translate(-12, 0, 0); rotateZ(radians(25)); box(26, 3, 7); popMatrix();
  pushMatrix(); translate(12, 0, 0); rotateZ(radians(-25)); box(26, 3, 7); popMatrix();
  fill(180, 105, 45); box(14, 4, 8);
  popMatrix();
}

void drawHeldValari() {
  pushMatrix(); translate(runnerX * PPM, -4.6 * PPM, runnerY * PPM); rotateY(-runnerHeading); 
  translate(-1, 0, 9); rotateX(radians(75)); rotateZ(radians(-15));
  drawBoomerang(); popMatrix();
}

void drawFlyingValari() {
  pushMatrix(); translate(pos.x*PPM, -pos.z*PPM, pos.y*PPM);
  float heading = atan2(vel.y, vel.x);
  float pitch = atan2(vel.z, max(0.1, sqrt(vel.x*vel.x + vel.y*vel.y)));
  rotateY(-heading); rotateZ(pitch);
  rotateX(sin(spinPhase)*0.35); rotateY(spinPhase);
  drawBoomerang(); popMatrix();
}

// ====================================================
// THROW LOGIC (DYNAMIC SLIDERS ON THE RUN)
// ====================================================

void throwValari() {
  if (isDead) { status = "He's dead. You can't throw."; return; }
  if (valariState == V_FLY) { status = "Valari is flying. Wait for return."; return; }
  launchNewThrow();
}

void launchNewThrow() {
  if (flight.size() > 1) {
    saved.add(new SavedFlight(flight, pathCol));
    while (saved.size() > MAX_SAVED) saved.remove(0);
  }

  float runV = horseState == H_RUN ? RUN_SPEED : 0;
  pathCol = color(random(55, 255), random(55, 220), random(75, 255));

  float flightTime = sliders.get(0).val;
  float inclination_i = sliders.get(1).val; 
  float CL_val = sliders.get(2).val;             
  float CD_val = sliders.get(3).val;           
  float alpha_v_val = 20.0; 

  launchHeading = runnerHeading; 
  if (scribble.size() > 1 && horseState != H_STOP) {
    V3 p0 = scribble.get(max(0, scribble.size()-2)); 
    V3 p1 = scribble.get(scribble.size()-1);
    launchHeading = atan2(p1.y - p0.y, p1.x - p0.x);
  }

  model = new ThrowModel(++throwCount, pathCol, inclination_i, CL_val, CD_val, alpha_v_val,
                         runnerX, runnerY, 4.6, BASE_SPEED + runV, runV, flightTime);
  
  pos = new V3(runnerX, runnerY, 4.6);
  flight.clear(); flight.add(new FlightPoint(pos.x, pos.y, pos.z, 0, model.initialSpeed));

  animTime = 0; spinPhase = 0; valariState = V_FLY; viewMode = MODE_3D;
  status = String.format("THROW #%d — Sliders locked in for this run!", throwCount);
}

void finishFlight() {
  if (valariState != V_FLY || model == null) return;
  valariState = V_RET;
  
  if (!isDead) {
    if (horseState == H_RUN && autoThrow) launchNewThrow(); 
    else status = "RETURNED TO HAND — Ready! Tap THROW again!";
  }
}

// ====================================================
// RIDE, STOP AND RESET
// ====================================================

void startRun() {
  if (isDead) return;
  horseState = H_RUN; viewMode = MODE_3D; 
  status = "RIDING FAST — Dodge the intense muskets & artillery!";
}

void stopRun() { if (!isDead) horseState = H_STOP; status = "HORSE STOPPED."; }

void resetExperiment() {
  flight.clear(); model = null; cannonballs.clear(); musketballs.clear(); scribble.clear(); particles.clear();
  animTime = 0; runnerX = 0; runnerY = 0; runnerHeading = 0; runDist = 0; runCycle = 0;
  isDead = false; horseState = H_READY; valariState = V_READY; viewMode = MODE_DRAW;
  spawnEnemies(); status = "ARMY ARRAY RESET — Draw a path to dodge them!";
}

// ====================================================
// HUD & INTERFACES
// ====================================================

void drawHUD() {
  camera(); hint(DISABLE_DEPTH_TEST); noLights();
  rectMode(CORNER); noStroke();
  fill(18, 28, 38, 245); rect(0, 0, width, height*0.13);

  button(rideX, btnY, rideW, btnH, "RIDE", isDead ? color(100) : color(30, 155, 80));
  button(stopX, btnY, stopW, btnH, "STOP", isDead ? color(100) : color(220, 125, 30));
  button(throwX, btnY, throwW, btnH, "THROW", isDead ? color(100) : color(30, 100, 220));
  button(drawX, btnY, drawW, btnH, "2D DRAW", color(100, 100, 115));
  button(viewX, btnY, viewW, btnH, "3D VIEW", color(110, 75, 180));
  button(autoX, btnY, autoW, btnH, autoThrow ? "AUTO: ON" : "AUTO: OFF", autoThrow ? color(30, 155, 80) : color(100)); 
  button(resetX, btnY, resetW, btnH, "RESET ARRAY", color(190, 45, 60));

  fill(isDead ? color(255,50,50) : color(255)); 
  textAlign(LEFT, CENTER); textSize(max(11, height*0.017));
  text("STATE: " + (isDead ? "OBLITERATED" : "ALIVE") + " | THROW: " + throwCount, width*0.02, height*0.925);
  textSize(max(10, height*0.016)); text(status, width*0.02, height*0.965);

  if (viewMode == MODE_3D) drawModelPanel();
  hint(ENABLE_DEPTH_TEST);
}

void drawModelPanel() {
  float x = width*0.72, y = height*0.15, w = width*0.275, h = height*0.72;
  noStroke(); fill(10, 18, 30, 225); rect(x, y, w, h, 10);
  stroke(130, 175, 205); noFill(); rect(x, y, w, h, 10);

  noStroke(); fill(255, 225, 90); textAlign(LEFT, TOP); textSize(max(11, height*0.018));
  text("VALARI PARAMETRIC MODEL", x+10, y+9);

  fill(220, 235, 245); textSize(max(9, height*0.014));
  float yy = y + height*0.055, gap = height*0.027;

  if (model == null) {
    fill(255, 210, 110); text("Tap THROW to launch Valari.", x+10, yy);
  } else {
    fill(120, 225, 255); text("THROW #" + model.id, x+10, yy); yy += gap;
    fill(240);
    text("Duration = " + nf(model.duration,1,2) + " s", x+10, yy); yy += gap;
    text("Max height = " + nf(model.maxHeight,1,2) + " m", x+10, yy); yy += gap;
    text("Loop scale = " + nf(model.range,1,2) + " m", x+10, yy);
  }
  
  for (UISlider s : sliders) s.draw();
}

void button(float x, float y, float w, float h, String label, int col) {
  noStroke(); fill(0, 0, 0, 75); rect(x+2, y+2, w, h, 7);
  fill(col); rect(x, y, w, h, 7);
  stroke(255, 170); noFill(); rect(x, y, w, h, 7);
  noStroke(); fill(255); textAlign(CENTER, CENTER); textSize(max(9, height*0.017));
  text(label, x+w/2, y+h/2);
}

// ====================================================
// TOUCH / MOUSE INPUT
// ====================================================

void mousePressed() {
  lastMX = mouseX; lastMY = mouseY;
  
  if (viewMode == MODE_3D) {
    for (UISlider s : sliders) {
      if (s.hit(mouseX, mouseY)) { activeSlider = s; activeSlider.drag(mouseX); return; }
    }
  }
  
  if (hit(mouseX, mouseY, rideX, btnY, rideW, btnH)) { startRun(); return; }
  if (hit(mouseX, mouseY, stopX, btnY, stopW, btnH)) { stopRun(); return; }
  if (hit(mouseX, mouseY, throwX, btnY, throwW, btnH)) { throwValari(); return; }
  if (hit(mouseX, mouseY, drawX, btnY, drawW, btnH)) { viewMode = MODE_DRAW; status = "2D DRAW: draw locus path for horse."; return; }
  if (hit(mouseX, mouseY, viewX, btnY, viewW, btnH)) { viewMode = MODE_3D; status = "3D VIEW: drag to rotate."; return; }
  if (hit(mouseX, mouseY, autoX, btnY, autoW, btnH)) { autoThrow = !autoThrow; return; } 
  if (hit(mouseX, mouseY, resetX, btnY, resetW, btnH)) { resetExperiment(); return; }
  
  if (viewMode == MODE_DRAW && mouseY > height*0.14) {
    if (scribble.size() >= MAX_SCRIBBLE) scribble.clear();
    scribble.add(screenToWorld(mouseX, mouseY));
    drawing = true;
  }
}

void mouseDragged() {
  if (activeSlider != null) { activeSlider.drag(mouseX); return; }
  
  if (viewMode == MODE_DRAW && drawing && mouseY > height*0.14) {
    if (scribble.size() < MAX_SCRIBBLE) {
      V3 p = screenToWorld(mouseX, mouseY);
      V3 prev = scribble.get(scribble.size()-1);
      if (distance2D(prev.x, prev.y, p.x, p.y) > 0.06) scribble.add(p);
    }
  } else if (viewMode == MODE_3D && mouseY > height*0.14 && mouseY < height*0.88) {
    camY += (mouseX-lastMX)*0.008;
    camX = constrain(camX-(mouseY-lastMY)*0.008, -PI/2, PI/2);
    lastMX = mouseX; lastMY = mouseY;
  }
}

void mouseReleased() { 
  drawing = false; activeSlider = null;
  if (viewMode == MODE_DRAW && scribble.size() > 0) {
    runDist = 0; runnerX = scribble.get(0).x; runnerY = scribble.get(0).y;
    if (scribble.size() > 1) {
       runnerHeading = atan2(scribble.get(1).y - scribble.get(0).y, scribble.get(1).x - scribble.get(0).x);
    }
  }
}

boolean hit(float px, float py, float x, float y, float w, float h) { return px >= x && px <= x+w && py >= y && py <= y+h; }
