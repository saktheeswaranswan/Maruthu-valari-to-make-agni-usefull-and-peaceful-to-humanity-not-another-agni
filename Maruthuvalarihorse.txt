/*
 TAMIL VALARI — 3D FLIGHT & PURE PARAMETRIC AERODYNAMICS (HORSEBACK RIDER EDITION)
 Processing Java Mode / Android APDE — P3D

 FEATURES
 1. 2D touch-drawn locus scribble to set launch heading & running path.
 2. Tap "RIDE" to automatically switch to 3D and watch the horse traverse the exact path.
 3. Rider on horseback smoothly turns to face the path tangent.
 4. Rotatable 3D ground and trajectory.
 5. Strict 3D parametric trajectory ensuring a perfect return to the rider's hand.
 6. Model coefficients displayed per throw.
 7. THROW button continuously blinks upon return, waiting for manual click.
*/

import java.util.ArrayList;

// -------------------- STATES --------------------

final int MODE_DRAW = 0;
final int MODE_3D = 1;

final int READY = 0;
final int RUNNING = 1;
final int STOPPED = 2;
final int FLYING = 3;
final int RETURNED = 4;

// -------------------- PARAMETERS --------------------

final float MIN_ANGLE = 5;
final float MAX_ANGLE = 85;

final float BASE_SPEED = 20.0;
final float RUN_SPEED = 5.0;
final float GRAVITY = 9.81;
final float PPM = 14.0;
final float DT = 1.0 / 120.0;

final float FLIGHT_TIME = 4.8;
final int MAX_POINTS = 1600;
final int MAX_SCRIBBLE = 600;
final int MAX_SAVED = 6;

// -------------------- GLOBAL STATE --------------------

int viewMode = MODE_DRAW;
int state = READY;

boolean drawing = false;
boolean sliderDrag = false;

float angleDeg = 75; // Used as Inclination (i)

float runnerX = 0;
float runnerY = 0;
float runnerHeading = 0; // The direction the horse is facing
float runDist = 0;       // Distance traveled along the drawn curve
float runCycle = 0;

float animTime = 0;
float spinPhase = 0;
float launchHeading = 0;

float camX = radians(28);
float camY = radians(-35);
float lastMX, lastMY;

int throwCount = 0;
int pathCol;
String status = "Draw a launch path for the horse, then tap RIDE.";

float btnY, btnH;
float rideX, rideW, stopX, stopW;
float throwX, throwW, drawX, drawW;
float viewX, viewW, autoX, autoW;
float resetX, resetW;
float sliderX, sliderW, sliderY, sliderH;

// -------------------- DATA CLASSES --------------------

class V3 {
  float x, y, z;

  V3(float X, float Y, float Z) {
    x = X;
    y = Y;
    z = Z;
  }
}

class FlightPoint {
  float x, y, z, t, speed;

  FlightPoint(float X, float Y, float Z,
              float T, float S) {
    x = X;
    y = Y;
    z = Z;
    t = T;
    speed = S;
  }
}

class ThrowModel {
  int id, col;

  // Aerodynamic coefficients
  float inclination; // i
  float CL;          // Lift coefficient
  float CD;          // Drag coefficient
  float alpha_v;     // Angle of attack

  float releaseX, releaseY, releaseZ;
  float initialSpeed, runSpeed;
  float duration, maxHeight, range;

  ThrowModel(int I, int C, float INC, float CL_val, float CD_val, float ALPHA,
             float X, float Y, float Z,
             float V, float RV) {
    id = I;
    col = C;
    
    inclination = INC;
    CL = CL_val;
    CD = CD_val;
    alpha_v = ALPHA;

    releaseX = X;
    releaseY = Y;
    releaseZ = Z;

    initialSpeed = V;
    runSpeed = RV;

    duration = FLIGHT_TIME;
    maxHeight = Z;
    range = 30.0; // Parametric loop radius scaling
  }
}

class SavedFlight {
  ArrayList<FlightPoint> points = new ArrayList<FlightPoint>();
  int col;

  SavedFlight(ArrayList<FlightPoint> src, int C) {
    col = C;
    for (FlightPoint p : src) {
      points.add(new FlightPoint(p.x, p.y, p.z, p.t, p.speed));
    }
  }
}

ArrayList<V3> scribble = new ArrayList<V3>();
ArrayList<FlightPoint> flight = new ArrayList<FlightPoint>();
ArrayList<SavedFlight> saved = new ArrayList<SavedFlight>();

ThrowModel model = null;

// The release height matches the rider's hand on horseback (approx 4.6m * 14ppm = ~64 pixels up)
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
}

void layoutUI() {
  btnY = height * 0.025;
  btnH = height * 0.09;

  rideX = width * 0.012;
  rideW = width * 0.082;
  stopX = width * 0.099;
  stopW = width * 0.082;
  throwX = width * 0.186;
  throwW = width * 0.095;
  drawX = width * 0.286;
  drawW = width * 0.091;
  viewX = width * 0.382;
  viewW = width * 0.091;
  autoX = width * 0.478;
  autoW = width * 0.086; // Spacer

  sliderX = width * 0.574;
  sliderW = width * 0.195;
  sliderY = btnY;
  sliderH = btnH;

  resetX = width * 0.85;
  resetW = width * 0.135;
}

// ====================================================
// MAIN LOOP
// ====================================================

void draw() {
  float dt = constrain(1.0 / max(frameRate, 1), 0, 0.05);
  updateFrame(dt);

  if (viewMode == MODE_DRAW) {
    drawOrthographic();
  } else {
    drawScene3D();
  }
  drawHUD();
}

void updateFrame(float dt) {
  if (state == RUNNING) {
    runCycle += dt * 10;
    if (scribble.size() > 1) {
      runDist += RUN_SPEED * dt;
      updateRunnerPosition();
    } else {
      runnerX += RUN_SPEED * dt;
      runnerHeading = 0; 
    }
  }

  if (state == FLYING) {
    animTime += dt;
    integrateFlight(dt);
  }
}

// ====================================================
// HORSE PATH INTERPOLATION
// ====================================================

void updateRunnerPosition() {
  if (scribble.size() < 2) return;
  
  float currentDist = 0;
  for (int i = 0; i < scribble.size() - 1; i++) {
    V3 p1 = scribble.get(i);
    V3 p2 = scribble.get(i + 1);
    float segLen = distance2D(p1.x, p1.y, p2.x, p2.y);
    
    if (currentDist + segLen >= runDist) {
      float t = (runDist - currentDist) / segLen;
      runnerX = lerp(p1.x, p2.x, t);
      runnerY = lerp(p1.y, p2.y, t);
      // Turn the horse perfectly tangent to the path segment
      runnerHeading = atan2(p2.y - p1.y, p2.x - p1.x);
      return;
    }
    currentDist += segLen;
  }
  
  // Reached the end of the curve
  V3 lastP = scribble.get(scribble.size() - 1);
  runnerX = lastP.x;
  runnerY = lastP.y;
  
  // Retain final direction
  if (scribble.size() > 1) {
    V3 p2 = scribble.get(scribble.size() - 2);
    runnerHeading = atan2(lastP.y - p2.y, lastP.x - p2.x);
  }
  
  state = STOPPED;
  status = "Horse reached the end of the curve.";
}

// ====================================================
// VECTOR HELPERS
// ====================================================

float speedOf(V3 v) {
  return sqrt(v.x * v.x + v.y * v.y + v.z * v.z);
}

float distance2D(float x1, float y1, float x2, float y2) {
  return sqrt(sq(x1 - x2) + sq(y1 - y2));
}

// ====================================================
// 3D PARAMETRIC RETURN TRAJECTORY 
// ====================================================

void integrateFlight(float dt) {
  if (model == null || state != FLYING) return;

  float progress = min(1.0, animTime / model.duration);
  
  // Mathematical 3D parametric trajectory ensuring perfect return to the rider's hand
  float theta = progress * TWO_PI;
  float R = model.range / 2.0; 
  float H = 4.5; // Z-axis altitude gain

  // Base parametric formulas for a returning curve
  float bx = R * sin(theta);
  float by = R * (1 - cos(theta));
  float bz = H * sin(theta / 2.0);

  // Apply inclination tilt (around local X axis) based on slider UI
  float tiltRad = radians(90 - model.inclination);
  float tiltY = by * cos(tiltRad) - bz * sin(tiltRad);
  float tiltZ = by * sin(tiltRad) + bz * cos(tiltRad);

  // Apply Launch Heading (yaw) from the curve direction
  float yaw = launchHeading;
  float tx = bx * cos(yaw) - tiltY * sin(yaw);
  float ty = bx * sin(yaw) + tiltY * cos(yaw);
  float tz = tiltZ;

  // Set absolute position returning perfectly to release coordinates
  pos.x = model.releaseX + tx;
  pos.y = model.releaseY + ty;
  pos.z = model.releaseZ + tz;

  // Calculate analytical derivatives for velocity to accurately angle the 3D model
  float w = TWO_PI / model.duration;
  float dbx = R * w * cos(theta);
  float dby = R * w * sin(theta);
  float dbz = H * (w / 2.0) * cos(theta / 2.0);

  float dtiltY = dby * cos(tiltRad) - dbz * sin(tiltRad);
  float dtiltZ = dby * sin(tiltRad) + dbz * cos(tiltRad);

  vel.x = dbx * cos(yaw) - dtiltY * sin(yaw);
  vel.y = dbx * sin(yaw) + dtiltY * cos(yaw);
  vel.z = dtiltZ;

  spinPhase += 25.0 * dt; // Visual spin rate
  
  model.maxHeight = max(model.maxHeight, pos.z);

  // Record point for trace (Trace perfectly overlays previous identical throws)
  if (flight.size() < MAX_POINTS && animTime <= model.duration) {
    flight.add(new FlightPoint(pos.x, pos.y, pos.z, animTime, speedOf(vel)));
  }

  // End conditions
  if (progress >= 1.0) {
    finishFlight();
  }
}

// ====================================================
// 2D DRAW VIEW
// ====================================================

float centerX() { return width * 0.43; }
float centerY() { return height * 0.57; }

float worldToScreenX(float x) { return centerX() + (x - runnerX) * PPM; }
float worldToScreenY(float y) { return centerY() - (y - runnerY) * PPM; }

V3 screenToWorld(float sx, float sy) {
  return new V3(runnerX + (sx - centerX()) / PPM,
                runnerY + (centerY() - sy) / PPM, 0);
}

void drawOrthographic() {
  background(220, 230, 233);
  noStroke(); fill(245);
  rect(0, height*0.13, width, height*0.87);

  stroke(190, 202, 208); strokeWeight(1);
  float step = 5 * PPM;
  for (float x = centerX() % step; x < width; x += step) line(x, height*0.14, x, height);
  for (float y = height*0.14; y < height; y += step) line(0, y, width, y);

  stroke(120, 140, 150); strokeWeight(2);
  line(0, centerY(), width, centerY());
  line(centerX(), height*0.14, centerX(), height);

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

  // Draw horse/rider top-down icon
  noStroke(); fill(100, 50, 20);
  ellipse(worldToScreenX(runnerX), worldToScreenY(runnerY), 22, 22);
  fill(255, 220, 60);
  ellipse(worldToScreenX(runnerX), worldToScreenY(runnerY), 8, 8);

  fill(25); textAlign(LEFT, TOP); textSize(14);
  text("2D TOP VIEW — Draw to set the horse locus path & launch direction.", 18, height*0.15 + 8);
  text("Path Points: " + scribble.size() + " | Grid spacing: 5 m", 18, height*0.15 + 28);
  text("Trajectory returns perfectly to the horseback rider using precise parametric curves.", 18, height*0.15 + 48);
}

// ====================================================
// 3D VIEW
// ====================================================

void drawScene3D() {
  background(150, 202, 230);
  pushMatrix();
  translate(width*0.40, height*0.78, -70);
  rotateX(camX); rotateY(camY);

  float focusX = runnerX, focusY = runnerY;
  if (model != null && (state == FLYING || state == RETURNED)) {
    focusX = model.releaseX; focusY = model.releaseY;
  }
  translate(-focusX*PPM, 0, -focusY*PPM);

  lights(); ambientLight(125, 125, 125);
  directionalLight(255, 255, 245, -0.5, 1, -0.5);

  drawGround(); drawSavedFlights3D(); drawScribble3D();
  drawCurrentFlight3D(); drawRiderOnHorse();

  if (state == FLYING) drawFlyingValari();
  else drawHeldValari();

  popMatrix();
}

void drawGround() {
  pushMatrix(); rotateX(PI/2); rectMode(CENTER);
  noStroke(); fill(110, 165, 85); rect(0, 0, 6000, 6000);
  stroke(255, 255, 255, 55); strokeWeight(1);
  float step = 5 * PPM;
  for (float i = -3000; i <= 3000; i += step) {
    line(i, -3000, i, 3000); line(-3000, i, 3000, i);
  }
  popMatrix();
}

void drawScribble3D() {
  if (scribble.size() < 2) return;
  noFill(); stroke(pathCol, 190); strokeWeight(2);
  beginShape();
  for (V3 p : scribble) vertex(p.x*PPM, -0.03*PPM, p.y*PPM);
  endShape();
}

void drawSavedFlights3D() {
  for (SavedFlight s : saved) {
    noFill(); stroke(s.col, 90); strokeWeight(2);
    beginShape();
    for (FlightPoint p : s.points) vertex(p.x*PPM, -p.z*PPM, p.y*PPM);
    endShape();
  }
}

void drawCurrentFlight3D() {
  if (flight.size() < 2) return;
  noFill(); stroke(model != null ? model.col : pathCol); strokeWeight(4);
  beginShape();
  for (FlightPoint p : flight) vertex(p.x*PPM, -p.z*PPM, p.y*PPM);
  endShape();
}

void drawRiderOnHorse() {
  pushMatrix();
  translate(runnerX * PPM, 0, runnerY * PPM);
  
  // Orient horse along the tangent of the scribble path
  rotateY(-runnerHeading); 

  float hBob = state == RUNNING ? abs(sin(runCycle * 1.5)) * 4 : 0;
  translate(0, -hBob, 0);

  // ---------- DRAW HORSE ----------
  fill(100, 50, 20); // Brown Body
  pushMatrix(); translate(0, -35, 0); box(40, 20, 20); popMatrix();
  
  // Neck/Head
  pushMatrix(); translate(15, -50, 0); rotateZ(radians(30)); box(12, 25, 10); popMatrix();
  pushMatrix(); translate(23, -58, 0); box(14, 10, 9); popMatrix(); // Snout
  
  // Legs Animation
  float hSwing = state == RUNNING ? sin(runCycle * 1.5) * 0.4 : 0;
  fill(60, 30, 15);
  
  // Front Left & Right
  pushMatrix(); translate(12, -20, -7); rotateZ(hSwing); translate(0, 10, 0); box(6, 22, 6); popMatrix();
  pushMatrix(); translate(12, -20, 7); rotateZ(-hSwing); translate(0, 10, 0); box(6, 22, 6); popMatrix();
  // Back Left & Right
  pushMatrix(); translate(-12, -20, -7); rotateZ(-hSwing); translate(0, 10, 0); box(6, 22, 6); popMatrix();
  pushMatrix(); translate(-12, -20, 7); rotateZ(hSwing); translate(0, 10, 0); box(6, 22, 6); popMatrix();

  // ---------- DRAW RIDER ----------
  translate(-4, -50, 0); // Positioned sitting on horse back
  
  // Torso
  fill(150, 85, 50);
  pushMatrix(); translate(0, -10, 0); box(12, 20, 14); popMatrix();
  // Head
  fill(70, 42, 28);
  pushMatrix(); translate(0, -26, 0); sphereDetail(10); sphere(7); popMatrix();
  
  // Arms
  fill(45, 45, 55);
  // Left arm
  pushMatrix(); translate(0, -15, -8); rotateZ(radians(10)); translate(0, 9, 0); box(5, 18, 6); popMatrix();
  // Right arm (Holding Valari - raised slightly)
  pushMatrix(); translate(0, -15, 8); rotateZ(state == FLYING ? radians(25) : radians(-15)); translate(0, 9, 0); box(5, 18, 6); popMatrix();
  
  // Legs (spread to sit on horse)
  fill(150, 85, 50);
  pushMatrix(); translate(-2, 2, -8); rotateX(radians(-35)); translate(0, 10, 0); box(5, 20, 5); popMatrix();
  pushMatrix(); translate(-2, 2, 8); rotateX(radians(35)); translate(0, 10, 0); box(5, 20, 5); popMatrix();

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
  pushMatrix();
  // Attach correctly to rider's right hand while on horseback
  translate(runnerX * PPM, -4.6 * PPM, runnerY * PPM);
  rotateY(-runnerHeading); 
  
  // Offset to match the rider's right hand exactly
  translate(-1, 0, 9); 
  rotateX(radians(75)); rotateZ(radians(-15));
  
  drawBoomerang();
  popMatrix();
}

void drawFlyingValari() {
  pushMatrix();
  translate(pos.x*PPM, -pos.z*PPM, pos.y*PPM);
  float heading = atan2(vel.y, vel.x);
  float pitch = atan2(vel.z, max(0.1, sqrt(vel.x*vel.x + vel.y*vel.y)));
  rotateY(-heading); rotateZ(pitch);
  rotateX(sin(spinPhase)*0.35); rotateY(spinPhase);
  drawBoomerang();
  popMatrix();
}

// ====================================================
// THROW LOGIC
// ====================================================

void throwValari() {
  if (state == FLYING) {
    status = "Valari is flying. Wait for completion.";
    return;
  }
  launchNewThrow();
}

void launchNewThrow() {
  float runV = state == RUNNING ? RUN_SPEED : 0;
  float speed = constrain(BASE_SPEED + runV, 8, 27);
  pathCol = color(random(55, 255), random(55, 220), random(75, 255));

  float inclination_i = angleDeg; // Derived directly from Slider UI
  float CL_val = 1.5;             
  float alpha_v_val = 20.0;
  float CD_val = 0.12;           

  // Elevated release point for horseback rider
  float releaseX = runnerX, releaseY = runnerY, releaseZ = 4.6;
  launchHeading = runnerHeading; // Follow horse direction

  if (scribble.size() > 1 && state != STOPPED) {
    // dynamically ensure throw direction matches current movement
    V3 p0 = scribble.get(max(0, scribble.size()-2)); 
    V3 p1 = scribble.get(scribble.size()-1);
    launchHeading = atan2(p1.y - p0.y, p1.x - p0.x);
  }

  model = new ThrowModel(++throwCount, pathCol, inclination_i, CL_val, CD_val, alpha_v_val,
                         releaseX, releaseY, releaseZ, speed, runV);

  pos = new V3(releaseX, releaseY, releaseZ);
  
  flight.clear();
  flight.add(new FlightPoint(pos.x, pos.y, pos.z, 0, speed));

  animTime = 0; spinPhase = 0;
  state = FLYING; viewMode = MODE_3D;
  
  status = String.format("THROW #%d — i:%.1f° Parametric Trace Active", throwCount, inclination_i);
}

void finishFlight() {
  if (state != FLYING || model == null) return;
  state = RETURNED;
  // Flashes endlessly until clicked
  status = "RETURNED TO HORSEBACK — Tap THROW again!";
}

// ====================================================
// RIDE, STOP AND RESET
// ====================================================

void startRun() {
  if (state == FLYING) { status = "Wait until the current flight finishes."; return; }
  state = RUNNING; 
  viewMode = MODE_3D; // Instantly switch to Next Screen 3D view as requested
  status = "RIDING — horse perfectly tracking drawn curve.";
}

void stopRun() {
  state = STOPPED; status = "HORSE STOPPED.";
}

void resetExperiment() {
  if (flight.size() > 1) {
    saved.add(new SavedFlight(flight, pathCol));
    while (saved.size() > MAX_SAVED) saved.remove(0);
  }
  flight.clear(); model = null;
  animTime = 0; 
  runnerX = 0; runnerY = 0; runnerHeading = 0; 
  runDist = 0; runCycle = 0;
  
  if(scribble.size() > 0) {
     runnerX = scribble.get(0).x;
     runnerY = scribble.get(0).y;
  }
  
  pos = new V3(0, 0, 4.6); vel = new V3(0, 0, 0);
  state = READY; viewMode = MODE_DRAW;
  status = "RESET — draw a locus path for the horse.";
}

String stateName() {
  if (state == READY) return "READY";
  if (state == RUNNING) return "RIDING";
  if (state == STOPPED) return "STOPPED";
  if (state == FLYING) return "FLYING";
  if (state == RETURNED) return "RETURNED";
  return "UNKNOWN";
}

// ====================================================
// HUD
// ====================================================

void drawHUD() {
  camera(); hint(DISABLE_DEPTH_TEST); noLights();
  rectMode(CORNER); noStroke();
  fill(18, 28, 38, 245); rect(0, 0, width, height*0.13);

  button(rideX, btnY, rideW, btnH, "RIDE", color(30, 155, 80));
  button(stopX, btnY, stopW, btnH, "STOP", color(220, 125, 30));
  
  // Continuously blink until clicked when returned
  int tc = color(30, 100, 220);
  if (state == RETURNED) {
     if ((millis() / 300) % 2 == 0) tc = color(255, 50, 50);
     else tc = color(255, 225, 35);
  }
  button(throwX, btnY, throwW, btnH, "THROW", tc);
  
  button(drawX, btnY, drawW, btnH, "2D DRAW", color(100, 100, 115));
  button(viewX, btnY, viewW, btnH, "3D VIEW", color(110, 75, 180));
  
  // Left blank as a spacer padding to preserve UI layout requested
  button(autoX, btnY, autoW, btnH, "MANUAL", color(90, 90, 100)); 
  
  button(resetX, btnY, resetW, btnH, "RESET", color(190, 45, 60));
  drawSlider();

  fill(255); textAlign(LEFT, CENTER); textSize(max(11, height*0.017));
  text("STATE: " + stateName() + " | INCLINATION (i): " + int(angleDeg) + "°" +
       " | THROW: " + throwCount,
       width*0.02, height*0.925);
  textSize(max(10, height*0.016));
  text(status, width*0.02, height*0.965);

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

  text("Gravity = " + nf(GRAVITY,1,2) + " m/s²", x+10, yy); yy += gap;
  text("Model = Parametric 3D Return Loop", x+10, yy); yy += gap;
  text("Path = Horseback to Hand Return", x+10, yy); yy += gap;
  text("Rider = Locked to User Locus Path", x+10, yy); yy += gap*1.2;

  if (model == null) {
    fill(255, 210, 110); text("Tap THROW to start.", x+10, yy);
  } else {
    fill(120, 225, 255); text("THROW #" + model.id, x+10, yy); yy += gap;

    fill(255, 215, 80);
    text("Inclination (i) = " + nf(model.inclination,1,1) + "°", x+10, yy); yy += gap;
    text("Lift Coeff (C_L) = " + nf(model.CL,1,2), x+10, yy); yy += gap;
    text("Drag Coeff (C_D) = " + nf(model.CD,1,2), x+10, yy); yy += gap;
    text("Angle of Attack (α_v) = " + nf(model.alpha_v,1,1) + "°", x+10, yy); yy += gap;

    fill(240);
    text("Release x = " + nf(model.releaseX,1,2) + " m", x+10, yy); yy += gap;
    text("Release y = " + nf(model.releaseY,1,2) + " m", x+10, yy); yy += gap;
    text("Duration = " + nf(model.duration,1,2) + " s", x+10, yy); yy += gap;
    text("Max height = " + nf(model.maxHeight,1,2) + " m", x+10, yy); yy += gap;
    text("Loop scale = " + nf(model.range,1,2) + " m", x+10, yy); yy += gap;
  }
}

void button(float x, float y, float w, float h, String label, int col) {
  noStroke(); fill(0, 0, 0, 75); rect(x+2, y+2, w, h, 7);
  fill(col); rect(x, y, w, h, 7);
  stroke(255, 170); noFill(); rect(x, y, w, h, 7);
  noStroke(); fill(255); textAlign(CENTER, CENTER); textSize(max(9, height*0.017));
  text(label, x+w/2, y+h/2);
}

// ====================================================
// ANGLE SLIDER (Controls Inclination `i`)
// ====================================================

void drawSlider() {
  noStroke(); fill(45, 55, 70); rect(sliderX, sliderY, sliderW, sliderH, 7);
  fill(255); textAlign(CENTER, CENTER); textSize(max(9, height*0.014));
  text("INCLINATION (i) " + int(angleDeg) + "°", sliderX+sliderW/2, sliderY+sliderH*0.22);
  float cy = sliderY + sliderH*0.70;
  stroke(220); strokeWeight(4); line(sliderX+10, cy, sliderX+sliderW-10, cy);
  float k = map(angleDeg, MIN_ANGLE, MAX_ANGLE, sliderX+10, sliderX+sliderW-10);
  noStroke(); fill(255, 215, 40); ellipse(k, cy, 16, 16);
}

// ====================================================
// TOUCH / MOUSE INPUT
// ====================================================

void mousePressed() {
  lastMX = mouseX; lastMY = mouseY;
  if (hit(mouseX, mouseY, sliderX, sliderY, sliderW, sliderH)) {
    sliderDrag = true; updateSlider(mouseX); return;
  }
  if (hit(mouseX, mouseY, rideX, btnY, rideW, btnH)) { startRun(); return; }
  if (hit(mouseX, mouseY, stopX, btnY, stopW, btnH)) { stopRun(); return; }
  if (hit(mouseX, mouseY, throwX, btnY, throwW, btnH)) { throwValari(); return; }
  if (hit(mouseX, mouseY, drawX, btnY, drawW, btnH)) { viewMode = MODE_DRAW; status = "2D DRAW: draw locus path for horse."; return; }
  if (hit(mouseX, mouseY, viewX, btnY, viewW, btnH)) { viewMode = MODE_3D; status = "3D VIEW: drag to rotate."; return; }
  if (hit(mouseX, mouseY, resetX, btnY, resetW, btnH)) { resetExperiment(); return; }
  
  if (viewMode == MODE_DRAW && mouseY > height*0.14) {
    if (state == FLYING) { status = "Wait for the current flight to finish."; return; }
    if (scribble.size() >= MAX_SCRIBBLE) scribble.clear();
    scribble.add(screenToWorld(mouseX, mouseY));
    drawing = true;
  }
}

void mouseDragged() {
  if (sliderDrag) { updateSlider(mouseX); return; }
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
  drawing = false; 
  sliderDrag = false;
  
  // Snap horse to the start of the drawn curve once finished
  if (viewMode == MODE_DRAW && scribble.size() > 0) {
    runDist = 0;
    runnerX = scribble.get(0).x;
    runnerY = scribble.get(0).y;
    // Pre-calculate initial facing direction
    if (scribble.size() > 1) {
       runnerHeading = atan2(scribble.get(1).y - scribble.get(0).y, scribble.get(1).x - scribble.get(0).x);
    }
  }
}

boolean hit(float px, float py, float x, float y, float w, float h) {
  return px >= x && px <= x+w && py >= y && py <= y+h;
}

void updateSlider(float x) {
  float u = constrain((x-(sliderX+10)) / max(1, sliderW-20), 0, 1);
  angleDeg = lerp(MIN_ANGLE, MAX_ANGLE, u);
}
