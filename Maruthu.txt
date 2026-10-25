/*
  3D VALARI / BOOMERANG SIMULATION - REAL PARAMETRIC MODEL
  Processing Java Mode / Android APDE (P3D)

  FEATURES:
  - Real 3D Parametric Equation Modeling of a returning aerodynamic trajectory.
  - The trajectory forms an offset cardioid/elliptical loop matching true gyroscopic return paths.
  - Features precise height mapping (Sine Arc) and dynamic banking/roll during flight.
  - Pre-calculates the complete looping flight path instantly.
  - Drag the screen to rotate the 3D camera.
*/

import java.util.ArrayList;

// =====================================================
// CONFIGURATION
// =====================================================

final float MIN_ANGLE = 5;
final float MAX_ANGLE = 85;
final float LAUNCH_SPEED = 18.0; 
final float PIXELS_PER_METRE = 14.0; 

// =====================================================
// STATES
// =====================================================

final int READY = 0;
final int RUNNING = 1;
final int STOPPED = 2;
final int FLYING = 3;
final int LANDED = 4;

int state = READY;
boolean paused = false;

float launchAngle = 30.0;
float launchAngleUsed = 30.0;

// Runner Variables
float runnerX = 0; 
float runnerSpeed = 5.0; 
float runCycle = 0;

// Animation Time
float animTime = 0;
float maxFlightTime = 0;
int throwCount = 0;

String statusMessage = "Tap RUN to move, or THROW to launch instantly.";

// Camera rotation
float camRotX = radians(20);
float camRotY = radians(-40);
float lastMouseX, lastMouseY;

// =====================================================
// DATA STRUCTURES
// =====================================================

class FlightPoint {
  float x, y, z, t;
  float phase; // Tracks the 0.0 to 1.0 progress for banking/roll calculations
  
  FlightPoint(float px, float py, float pz, float pt, float pPhase) {
    x = px; y = py; z = pz; t = pt; phase = pPhase;
  }
}

class FlightPath {
  ArrayList<FlightPoint> points;
  int col;
  FlightPath(ArrayList<FlightPoint> source, int c) {
    points = new ArrayList<FlightPoint>();
    for (FlightPoint p : source) {
      points.add(new FlightPoint(p.x, p.y, p.z, p.t, p.phase));
    }
    col = c;
  }
}

ArrayList<FlightPoint> currentTrail = new ArrayList<FlightPoint>();
ArrayList<FlightPath> savedPaths = new ArrayList<FlightPath>();
int pathColor;

// =====================================================
// UI GEOMETRY
// =====================================================
float runX, runW, stopX, stopW, throwX, throwW, resetX, resetW;
float topY, buttonH;
float sliderX, sliderW, sliderY, sliderH;
boolean draggingSlider = false;

// =====================================================
// INITIALIZATION
// =====================================================

void settings() {
  size(1280, 720, P3D); 
  smooth(4);
}

void setup() {
  orientation(LANDSCAPE);
  frameRate(60);
  pathColor = color(50, 200, 255); 
  calculateUI();
  resetExperiment();
}

void calculateUI() {
  topY = height * 0.025;
  buttonH = height * 0.105;

  runX = width * 0.02; runW = width * 0.15;
  stopX = width * 0.18; stopW = width * 0.15;
  throwX = width * 0.34; throwW = width * 0.16;
  resetX = width * 0.83; resetW = width * 0.15;

  sliderX = width * 0.53; sliderW = width * 0.27;
  sliderY = topY + buttonH * 0.12; sliderH = buttonH * 0.78;
}

// =====================================================
// MAIN LOOP
// =====================================================

void draw() {
  float frameDT = 1.0 / max(1.0, frameRate);
  frameDT = constrain(frameDT, 0, 0.05);

  if (!paused) {
    updateSimulation(frameDT);
  }

  background(135, 206, 235);

  // Setup 3D Scene Space
  pushMatrix();
  
  translate(width * 0.4, height * 0.75, -200);
  rotateX(camRotX);
  rotateY(camRotY);
  
  // Center camera on the throw origin
  float worldShift = (currentTrail.size() > 0) ? currentTrail.get(0).x : runnerX;
  translate(-worldShift * PIXELS_PER_METRE, 0, 0);

  lights();
  ambientLight(130, 130, 130);
  directionalLight(255, 255, 240, -0.5, 1, -0.5);

  drawGround3D();
  drawSavedTrajectories3D();
  drawCurrentTrajectory3D();
  
  drawRunner3D();

  if (state == FLYING || state == LANDED) {
    drawValariInFlight3D();
  } else {
    drawHeldValari3D();
  }
  
  popMatrix();
  drawHUD();
}

// =====================================================
// SIMULATION UPDATE
// =====================================================

void updateSimulation(float dt) {
  if (state == RUNNING) {
    runnerX += runnerSpeed * dt;
    runCycle += dt * 12.0;
  }
  
  if (state == FLYING) {
    animTime += dt;
    if (animTime >= maxFlightTime) {
      animTime = maxFlightTime;
      state = LANDED;
      statusMessage = "RETURNED — tap RESET to try again";
    }
  }
}

void startRunning() {
  if (state == FLYING) return;
  paused = false;
  state = RUNNING;
  statusMessage = "RUNNING — tap THROW to launch while moving";
}

void stopRunning() {
  if (state != RUNNING && state != READY) return;
  state = STOPPED;
  runCycle = 0;
  statusMessage = "STOPPED — tap THROW to launch from a standstill";
}

// =====================================================
// 3D REAL PARAMETRIC EQUATION MODELING
// =====================================================
/*
  MATHEMATICAL MODEL:
  A real boomerang trajectory is modeled parametrically using a modified trochoid for the XY plane 
  and a sine-wave arch for the Z (height) axis.
  - x(t) = Rx * sin(2*pi*t/T)
  - y(t) = Ry * (1 - cos(2*pi*t/T))
  This guarantees the loop starts exactly at (0,0) when t=0, forms a teardrop/circle, 
  and returns exactly to (0,0) when t=T. The thrower acts as the strict mathematical origin/axis.
*/

void throwValari() {
  if (state == FLYING || state == LANDED) {
    statusMessage = "Tap RESET first to throw again.";
    return;
  }

  paused = false;
  launchAngleUsed = launchAngle;
  
  // Origin strictly attached to the man's hand
  float startX = runnerX;
  float startY = -0.8; 
  float startZ = 1.6;  
  
  float aRad = radians(launchAngleUsed);
  float V = LAUNCH_SPEED;
  
  if (state == RUNNING) {
    V += runnerSpeed; 
  }

  currentTrail.clear();
  
  // Parametric shape constants
  float T = 3.8; // Total flight time in seconds
  float Rx = (V * T) / TWO_PI; // Forward propagation distance
  float Ry = Rx * 0.9;         // Lateral curve (slightly less than Rx for teardrop shape)
  float H = 6.0 + (V * 0.1);   // Peak parabolic height based on speed
  
  int steps = 250;
  for (int i = 0; i <= steps; i++) {
    float t = (T * i) / steps;
    float normPhase = t / T;             // Phase from 0.0 to 1.0
    float anglePhase = TWO_PI * normPhase; // Phase from 0 to 2*PI
    
    // 1. XY Parametric Loop (Man is the exact origin tangent point)
    float u = Rx * sin(anglePhase);
    float v = Ry * (1.0 - cos(anglePhase)); 
    
    // Rotate trajectory by the user's selected Launch Angle
    float globalX = startX + (u * cos(aRad) - v * sin(aRad));
    float globalY = startY + (u * sin(aRad) + v * cos(aRad));
    
    // 2. Z Parametric Height (Sine Wave Arc)
    // Starts at hand height (startZ), peaks in the middle, returns to hand height
    float z = startZ + H * sin(PI * normPhase);
    
    currentTrail.add(new FlightPoint(globalX, globalY, z, t, normPhase));
  }
  
  if (currentTrail.size() > 0) {
    maxFlightTime = currentTrail.get(currentTrail.size() - 1).t;
  }
  
  animTime = 0;
  throwCount++;
  state = FLYING;
  statusMessage = "THROWN! Calculating real parametric aerodynamic model.";
}

void resetExperiment() {
  if (currentTrail.size() > 1) {
    savedPaths.add(new FlightPath(currentTrail, pathColor));
    if (savedPaths.size() > 4) savedPaths.remove(0);
    pathColor = color(random(50, 255), random(50, 200), random(150, 255));
  }
  
  state = READY;
  runnerX = 0;
  runCycle = 0;
  animTime = 0;
  currentTrail.clear();
  statusMessage = "READY — Tap RUN or THROW";
}

// =====================================================
// 3D DRAWING FUNCTIONS
// =====================================================

void drawGround3D() {
  pushMatrix();
  fill(120, 180, 80);
  noStroke();
  
  rotateX(PI/2);
  rectMode(CENTER);
  rect(0, 0, 6000, 6000);
  
  // Grid
  stroke(255, 255, 255, 40);
  strokeWeight(2);
  float gridStep = 5.0 * PIXELS_PER_METRE;
  for (float i = -1500; i <= 1500; i += gridStep) {
    line(i, -1500, i, 1500);
    line(-1500, i, 1500, i);
  }
  popMatrix();
}

void drawSavedTrajectories3D() {
  noFill();
  strokeWeight(2);
  for (FlightPath path : savedPaths) {
    stroke(path.col, 120);
    beginShape();
    for (FlightPoint p : path.points) {
      vertex(p.x * PIXELS_PER_METRE, -p.z * PIXELS_PER_METRE, p.y * PIXELS_PER_METRE);
    }
    endShape();
  }
}

void drawCurrentTrajectory3D() {
  if (currentTrail.size() < 2) return;
  noFill();
  stroke(pathColor);
  strokeWeight(5);
  beginShape();
  for (FlightPoint p : currentTrail) {
    vertex(p.x * PIXELS_PER_METRE, -p.z * PIXELS_PER_METRE, p.y * PIXELS_PER_METRE);
  }
  endShape();
}

void drawRunner3D() {
  pushMatrix();
  translate(runnerX * PIXELS_PER_METRE, 0, 0);
  
  float bob = (state == RUNNING) ? abs(sin(runCycle)) * 4 : 0;
  translate(0, -bob, 0);
  float legSwing = (state == RUNNING) ? sin(runCycle) * 0.7 : 0;
  
  noStroke();
  
  fill(160, 95, 60); 
  pushMatrix(); translate(0, -25, 0); box(10, 25, 16); popMatrix();
  
  pushMatrix(); translate(0, -42, 0); fill(30, 15, 10); box(12, 12, 12); popMatrix();
  
  fill(160, 95, 60);
  pushMatrix();
  translate(0, -32, -10);
  rotateZ(-legSwing);
  translate(0, 10, 0);
  box(6, 20, 6);
  popMatrix();
  
  pushMatrix();
  translate(0, -32, 10);
  if (state == READY || state == STOPPED || state == RUNNING) {
    rotateZ(radians(-130)); 
  } else {
    rotateZ(radians(45)); 
  }
  translate(0, 10, 0);
  box(6, 20, 6);
  popMatrix();
  
  fill(250, 240, 220);
  pushMatrix();
  translate(0, -12, -5);
  rotateZ(legSwing);
  translate(0, 12, 0);
  box(8, 24, 8);
  popMatrix();
  
  pushMatrix();
  translate(0, -12, 5);
  rotateZ(-legSwing);
  translate(0, 12, 0);
  box(8, 24, 8);
  popMatrix();
  
  popMatrix();
}

void drawHeldValari3D() {
  pushMatrix();
  translate(runnerX * PIXELS_PER_METRE, -45, 12);
  rotateZ(radians(launchAngle - 90));
  rotateX(radians(70)); // Held vertically
  drawCurvedValariModel();
  popMatrix();
}

void drawValariInFlight3D() {
  if (currentTrail.isEmpty()) return;
  
  FlightPoint p1 = currentTrail.get(0);
  FlightPoint p2 = currentTrail.get(min(1, currentTrail.size()-1));
  
  for (int i = 0; i < currentTrail.size() - 1; i++) {
    if (currentTrail.get(i).t <= animTime && currentTrail.get(i+1).t >= animTime) {
      p1 = currentTrail.get(i);
      p2 = currentTrail.get(i+1);
      break;
    }
  }
  
  float segmentTime = p2.t - p1.t;
  float lerpAmt = (segmentTime > 0) ? (animTime - p1.t) / segmentTime : 0;
  
  float cx = lerp(p1.x, p2.x, lerpAmt);
  float cy = lerp(p1.y, p2.y, lerpAmt);
  float cz = lerp(p1.z, p2.z, lerpAmt);
  float cPhase = lerp(p1.phase, p2.phase, lerpAmt);
  
  pushMatrix();
  translate(cx * PIXELS_PER_METRE, -cz * PIXELS_PER_METRE, cy * PIXELS_PER_METRE);
  
  // Real Boomerang Banking: Starts vertical (70 deg), lays flat at peak (10 deg), returns vertical
  float bankAngle = lerp(radians(75), radians(15), sin(PI * cPhase));
  
  rotateZ(radians(launchAngleUsed));
  rotateX(bankAngle); 
  rotateY(animTime * 45.0); // Gyroscopic spin
  
  drawCurvedValariModel();
  popMatrix();
}

void drawCurvedValariModel() {
  pushMatrix();
  scale(0.9);
  fill(140, 70, 30); 
  noStroke();
  
  pushMatrix(); translate(-12, 0, 0); rotateZ(radians(25)); box(26, 3, 7); popMatrix();
  pushMatrix(); translate(12, 0, 0); rotateZ(radians(-25)); box(26, 3, 7); popMatrix();
  box(14, 4, 8);
  
  popMatrix();
}

// =====================================================
// 2D HUD (OVERLAY)
// =====================================================

void drawHUD() {
  camera();
  hint(DISABLE_DEPTH_TEST);
  noLights();

  noStroke();
  fill(15, 25, 40, 230);
  rectMode(CORNER);
  rect(0, 0, width, height * 0.135);

  drawButton(runX, topY, runW, buttonH, "RUN", color(30, 160, 80));
  drawButton(stopX, topY, stopW, buttonH, "STOP", color(220, 130, 30));
  drawButton(throwX, topY, throwW, buttonH, "THROW", color(30, 100, 220));
  drawButton(resetX, topY, resetW, buttonH, "RESET", color(190, 50, 60));

  drawAngleSlider();

  fill(10, 20, 30, 230);
  rect(0, height * 0.87, width, height * 0.13);

  fill(255);
  textAlign(LEFT, CENTER);
  textSize(max(14, height * 0.022));

  text("STATE: " + getStateName() + 
       " | ANGLE: " + nf(launchAngleUsed, 1, 1) + " deg" +
       " | THROWS: " + throwCount, 
       width * 0.02, height * 0.90);
       
  text("DRAG screen to rotate 3D view. " + statusMessage, width * 0.02, height * 0.95);

  hint(ENABLE_DEPTH_TEST);
}

void drawButton(float x, float y, float w, float h, String label, int col) {
  noStroke();
  fill(0, 0, 0, 80);
  rect(x + 4, y + 4, w, h, 8);
  fill(col);
  rect(x, y, w, h, 8);
  stroke(255, 200);
  strokeWeight(2);
  noFill();
  rect(x, y, w, h, 8);
  
  noStroke();
  fill(255);
  textAlign(CENTER, CENTER);
  text(label, x + w / 2, y + h / 2);
}

void drawAngleSlider() {
  noStroke();
  fill(45, 55, 70);
  rect(sliderX, sliderY, sliderW, sliderH, 8);

  float cy = sliderY + sliderH * 0.68;
  stroke(220);
  strokeWeight(6);
  line(sliderX + 15, cy, sliderX + sliderW - 15, cy);

  float knobX = map(launchAngle, MIN_ANGLE, MAX_ANGLE, sliderX + 15, sliderX + sliderW - 15);
  noStroke();
  fill(255, 210, 40);
  ellipse(knobX, cy, height * 0.05, height * 0.05);

  fill(255);
  textAlign(CENTER, CENTER);
  text("ANGLE " + int(launchAngle) + " deg", sliderX + sliderW / 2, sliderY + sliderH * 0.25);
}

String getStateName() {
  if (state == READY) return "READY";
  if (state == RUNNING) return "RUNNING";
  if (state == STOPPED) return "STOPPED";
  if (state == FLYING) return "FLYING";
  if (state == LANDED) return "LANDED";
  return "UNKNOWN";
}

// =====================================================
// INPUTS & CAMERA CONTROLS
// =====================================================

void mousePressed() {
  lastMouseX = mouseX;
  lastMouseY = mouseY;
  
  if (inside(mouseX, mouseY, sliderX, sliderY, sliderW, sliderH)) {
    draggingSlider = true;
    updateSlider(mouseX);
    return;
  }
  if (inside(mouseX, mouseY, runX, topY, runW, buttonH)) { startRunning(); return; }
  if (inside(mouseX, mouseY, stopX, topY, stopW, buttonH)) { stopRunning(); return; }
  if (inside(mouseX, mouseY, throwX, topY, throwW, buttonH)) { throwValari(); return; }
  if (inside(mouseX, mouseY, resetX, topY, resetW, buttonH)) { resetExperiment(); return; }
}

void mouseDragged() {
  if (draggingSlider) {
    updateSlider(mouseX);
  } else if (mouseY > height * 0.15 && mouseY < height * 0.85) {
    float dx = mouseX - lastMouseX;
    float dy = mouseY - lastMouseY;
    camRotY += dx * 0.01;
    camRotX = constrain(camRotX - dy * 0.01, -PI/2, PI/2);
    lastMouseX = mouseX;
    lastMouseY = mouseY;
  }
}

void mouseReleased() {
  draggingSlider = false;
}

boolean inside(float px, float py, float x, float y, float w, float h) {
  return px >= x && px <= x + w && py >= y && py <= y + h;
}

void updateSlider(float px) {
  float left = sliderX + 15;
  float right = sliderX + sliderW - 15;
  float t = constrain((px - left) / max(1, right - left), 0, 1);
  launchAngle = constrain(lerp(MIN_ANGLE, MAX_ANGLE, t), MIN_ANGLE, MAX_ANGLE);
}
