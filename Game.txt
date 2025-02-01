//Graphics preferences
rectMode(CENTER);
textAlign(CENTER, CENTER);
textFont(createFont("times new roman Bold"));
noStroke();

//Rendering variables
var minimapIsDrawn = false;
var viewAngle = 80;
var resolution = 200;
var viewRange = 600;
var flashLightR = 150;

//Scene and scene transition variables
var scene = "Menu";
var transition = false;
var transitionNum = 0;
var transitionStep = 0;
var nextScene = "";

//Graphics variables
var fogColour = color(0, 0, 0);
var wallImage = getImage("space/background");
var frameImage;
var enemyImage = getImage("avatars/orange-juice-squid");
var winImage = getImage("space/star");
var wallTexture = [];
var frameTexture = [];
var enemyTexture = [];
var winTexture = [];
var walls = [];
var cameraPos = [];
var wallsInRange = [];
var wallsInView = [];
var rays = [];

// Maze generation variables
var wallHeight = 100;
var wallWidth = 5;
var mazeSize = 10;
var mapSize = 1000;
var winX, winY, winAngle;

//Other
var gameOver = false;
var jumpscare = false;
var jumpscareTimer = 0;
var jumpscareImage;
var keyPress = [];


//Player variables
var player = {
    x: -mapSize/2 + mapSize/mazeSize/2,
    y: -mapSize/2 + mapSize/mazeSize/2,
    angle: 45,
    r: 10,
    speed: 1.5,
    rotSpeed: 2,
    isSprinting: false,
    maxStamina: 80,
    stamina: 80,
    moveTimer: 5,
};

//Enemy variables
var enemy = {
    active: false,
    x: 0,
    y: 0,
    angle: 0,
    width: 50,
    speed: 5,
    dist: 0,
    spawnTimer: millis(),
    spawnInterval: 20,
    startDelay: 20,
    spawnDist: 650
};


var startTransition = function(scene) {
    if (transition === false) {
        transition = true;
        transitionNum = 0;
        transitionStep = 0;
        nextScene = scene;
    }
};

//Useful functions for calculations
var findAngle = function(x, y, targetX, targetY) {
    return(atan2(targetY - y, targetX - x));
};
var findIntersection = function(line1, line2) {
    var xdiff = [line1[0][0] - line1[1][0], line2[0][0] - line2[1][0]];
    var ydiff = [line1[0][1] - line1[1][1], line2[0][1] - line2[1][1]];
    
    var det = function(a, b) {
        return a[0] * b[1] - a[1] * b[0];
    };
    
    var div = det(xdiff, ydiff);
    if (div === 0) {
    //   println('lines do not intersect');
    }
    
    var d = [det(line1[0], line1[1]), det(line2[0], line2[1])];
    var x = det(d, xdiff) / div;
    var y = det(d, ydiff) / div;
    
    return([x, y]);
};
var isLineIntersecting = function(line1, line2) {
    var x = line1[0][0];
    var y = line1[0][1];
    var x2 = line1[1][0];
    var y2 = line1[1][1];
    var tx = line2[0][0];
    var ty = line2[0][1];
    var tx2 = line2[1][0];
    var ty2 = line2[1][1];
    var num1 = (x2 - x) * (ty - y2) - (y2 - y) * (tx - x2);
    var num2 = (x2 - x) * (ty2 - y2) - (y2 - y) * (tx2 - x2);
    var num3 = (tx2 - tx) * (y - ty2) - (ty2 - ty) * (x - tx2);
    var num4 = (tx2 - tx) * (y2 - ty2) - (ty2 - ty) * (x2 - tx2);
    
    // var product = num1 * num2;
    // var product2 = num3 * num4;
    var pol1, pol2;
    if (num1 < 0 && num2 > 0 || num1 > 0 && num2 < 0) {
        pol1 = -1;
    } else {
        pol1 = 1;
    }
    if (num3 < 0 && num4 > 0 || num3 > 0 && num4 < 0) {
        pol2 = -1;
    } else {
        pol2 = 1;
    }
    
    return(pol1 < 0 && pol2 < 0);
    
};

//Represents each section of the maze (for maze generating)
var MazeQuad = function(points, gridX, gridY, walls) {
    this.points = points;
    this.gridX = gridX;
    this.gridY = gridY;
    this.walls = walls;
    
    this.wallPos = [];
    for (var i = 0; i < 4; i ++) {
        this.wallPos.push([points[i][0], points[i][1], points[(i+1)%4][0], points[(i+1)%4][1]]);
    }
};

//Wall object!
var Wall = function(x, y, x2, y2, image) {
    this.x = x;
    this.y = y;
    this.x2 = x2;
    this.y2 = y2;
    this.image = image;
    this.xAvg = (x + x2) / 2;
    this.yAvg = (y + y2) / 2;
    
    // this.num = walls.length;
    this.len = dist(this.x, this.y, this.x2, this.y2);
};
Wall.prototype.draw = function() {
    stroke(0, 0, 0);
    strokeWeight(2);
    line(this.x, this.y, this.x2, this.y2);
    strokeWeight(1);
    noStroke();
};
Wall.prototype.playerCollide = function() {
    var dot = (((player.x - this.x) * (this.x2 - this.x)) + ((player.y - this.y) * (this.y2 - this.y))) / sq(this.len);
    
    this.closestX = constrain(lerp(this.x, this.x2, dot), min(this.x, this.x2), max(this.x, this.x2));
    this.closestY = constrain(lerp(this.y, this.y2, dot), min(this.y, this.y2), max(this.y, this.y2));
    
    if (dist(this.closestX, this.closestY, player.x, player.y) < player.r) {
        if (this.image === winTexture) {
            startTransition("Win");
        } else {
            var collisionAngle = findAngle(this.closestX, this.closestY, player.x, player.y);
            player.x = this.closestX + cos(collisionAngle) * player.r;
            player.y = this.closestY + sin(collisionAngle) * player.r;
        }
    }
};
Wall.prototype.isInRange = function() {
    if (dist(player.x, player.y, this.xAvg, this.yAvg) < viewRange + this.len / 2) {
        return(true);
    }
    return(false);
};
Wall.prototype.isInView = function() {
    var angle1 = findAngle(player.x, player.y, this.x, this.y);
    var angle2 = findAngle(player.x, player.y, this.x2, this.y2);
    var maxAngle = max(angle1, angle2);
    var minAngle = min(angle1, angle2);
    if (maxAngle - minAngle > 180) {
        var curAngle = maxAngle;
        maxAngle = minAngle + 360;
        minAngle = curAngle;
    }
    var playerViewAngle = player.angle % 360;
    if (playerViewAngle - viewAngle/2 < maxAngle && playerViewAngle + viewAngle/2 > minAngle || playerViewAngle - viewAngle/2 < maxAngle + 360 && playerViewAngle + viewAngle/2 > minAngle + 360 || playerViewAngle - viewAngle/2 < maxAngle - 360 && playerViewAngle + viewAngle/2 > minAngle - 360) {
        return(true);
    }
};

//Ray object for raycasting
var Ray = function(x, y, angle) {
    this.x = x;
    this.y = y;
    this.angle = angle;
    
    this.x2 = x + cos(angle) * 99999;
    this.y2 = y + sin(angle) * 99999;
};
Ray.prototype.findClosestPoint = function() {
    var closestDist = 99999;
    var closestPos = [];
    var closestWallNum = 0;
    
    for (var i = 0; i < wallsInView.length; i ++) {
        var curWall = wallsInView[i];
        
        if (isLineIntersecting([[this.x, this.y], [this.x2, this.y2]], [[curWall.x, curWall.y], [curWall.x2, curWall.y2]])) {
            var intersectPos = findIntersection([[this.x, this.y], [this.x2, this.y2]], [[curWall.x, curWall.y], [curWall.x2, curWall.y2]]);
            
            var curDist = dist(player.x, player.y, intersectPos[0], intersectPos[1]);
            if (curDist < closestDist) {
                closestDist = curDist;
                closestPos = intersectPos;
                closestWallNum = i;
            }
        }
    }
    
    var closestWall = wallsInView[closestWallNum];
    var wallDist = dist(closestWall.x, closestWall.y, closestWall.x2, closestWall.y2);
    var pointDist = dist(closestPos[0], closestPos[1], closestWall.x, closestWall.y);
    return([closestPos, pointDist / wallDist, closestWall.image]);
};

//Load textures
background(fogColour);
image(wallImage, 0, 0, 100, wallHeight);
for (var i = 0; i < 100; i ++) {
    wallTexture.push(get(i, 0, 1, wallHeight));
}

background(0, 0, 50);
for (var i = 0; i < 100; i ++) {
    frameTexture.push(get(i, 0, 1, wallHeight));
}

background(fogColour);
image(enemyImage, -25, 0, 150, wallHeight);
for (var i = 0; i < 100; i ++) {
    enemyTexture.push(get(i, 0, 1, wallHeight));
}

background(255, 255, 0);
image(winImage, 0, 0, 100, wallHeight);
for (var i = 0; i < 100; i ++) {
    winTexture.push(get(i, 0, 1, wallHeight));
}

//Generates the maze
var generateMaze = function() {
    //Maze generation variables
    var mazePoints = [];
    var mazeQuads = [];
    var removerPos = [0, 0];
    var visited = [];
    var visitedCount = 0;
    var path = [[0, 0]];
    var possible = [];
    
    //Generate maze points and maze sections
    for (var i = 0; i < mazeSize + 1; i ++) {
        mazePoints.push([]);
        for (var j = 0; j < mazeSize + 1; j ++) {
            mazePoints[i].push([(j - mazeSize/2 + cos(random(0, 360)) * 0.35) * (mapSize/mazeSize), (i - mazeSize/2 + sin(random(0, 360)) * 0.35) * (mapSize/mazeSize)]);
        }
    }
    for (var i = 0; i < mazeSize; i ++) {
        mazeQuads.push([]);
        for (var j = 0; j < mazeSize; j ++) {
            mazeQuads[i].push(new MazeQuad([[mazePoints[i][j][0]+wallWidth, mazePoints[i][j][1]+wallWidth], [mazePoints[i][j+1][0]-wallWidth, mazePoints[i][j+1][1]+wallWidth], [mazePoints[i+1][j+1][0]-wallWidth, mazePoints[i+1][j+1][1]-wallWidth], [mazePoints[i+1][j][0]+wallWidth, mazePoints[i+1][j][1]-wallWidth]], j, i, [true, true, true, true]));
        }
    }
    
    //Mark each section as not visited
    for (var i = 0; i < mazeSize; i ++) {
        visited.push([]);
        for (var j = 0; j < mazeSize; j ++) {
            visited[i].push(false);
        }
    }
    
    //Function for adding possible moves
    var addPossible = function(x, y) {
        if (x >= 0 && x < mazeSize && y >= 0 && y < mazeSize) {
            if (visited[y][x] === false) {
                possible.push([x, y]);
            }
        }
    };
    
    //Maze generating loop
    while(true) {
        possible = [];
        addPossible(removerPos[0] + 1, removerPos[1]);
        addPossible(removerPos[0], removerPos[1] + 1);
        addPossible(removerPos[0] - 1, removerPos[1]);
        addPossible(removerPos[0], removerPos[1] - 1);
        
        if (visitedCount >= mazeSize * mazeSize) {
            break;
        }
        
        //If there are no possible moves, backtrack
        if (possible.length === 0) {
            removerPos = path[path.length-1];
            path.pop();
            
        //Otherwise, pick a random move and move there
        } else {
            var choice = possible[floor(random(0, possible.length))];
            var x = choice[0];
            var y = choice[1];
            path.push([x, y]);
            
            //Remove the walls as you move
            var wallDir;
            if (y < removerPos[1]) {
                wallDir = 0;
            } else if (x > removerPos[0]) {
                wallDir = 1;
            } else if (y > removerPos[1]) {
                wallDir = 2;
            } else if (x < removerPos[0]) {
                wallDir = 3;
            }
            var thisWall = mazeQuads[removerPos[1]][removerPos[0]];
            var thatWall = mazeQuads[y][x];
            thisWall.walls[wallDir] = false;
            thatWall.walls[(wallDir + 2) % 4] = false;
            walls.push(new Wall(thisWall.wallPos[wallDir][0], thisWall.wallPos[wallDir][1], thatWall.wallPos[(wallDir + 2) % 4][2], thatWall.wallPos[(wallDir + 2) % 4][3], frameTexture));
            walls.push(new Wall(thisWall.wallPos[wallDir][2], thisWall.wallPos[wallDir][3], thatWall.wallPos[(wallDir + 2) % 4][0], thatWall.wallPos[(wallDir + 2) % 4][1], frameTexture));
            
            //Mark the section as visited
            removerPos = [x, y];
            visited[y][x] = true;
            visitedCount += 1;
        }
    }
    
    for (var i = 0; i < mazeSize; i ++) {
        for (var j = 0; j < mazeSize; j ++) {
            for (var k = 0; k < 4; k ++) {
                if (mazeQuads[i][j].walls[k]) {
                    var curWall = mazeQuads[i][j].wallPos[k];
                    walls.push(new Wall(curWall[0], curWall[1], curWall[2], curWall[3], wallTexture));
                }
            }
        }
    }
    
    //Create the exit
    var winWall = walls[walls.length-2];
    winWall.image = winTexture;
    winX = winWall.xAvg;
    winY = winWall.yAvg;
};

var resetMaze = function() {
    player.x = -mapSize/2 + mapSize/mazeSize/2;
    player.y = -mapSize/2 + mapSize/mazeSize/2;
    player.angle = 45;
    
    enemy.active = false;
    enemy.spawnTimer = millis();
    enemy.startDelay = 20;
    
    gameOver = false;
    rays = [];
    walls = [];
    cameraPos = [];
    wallsInRange = [];
    wallsInView = [];
    
    
    generateMaze();
};

resetMaze();

draw = function() {
    if (scene === "Menu") {
        var bgVal = noise(millis() / 4000);
        background(0, 50 * bgVal, 100 * bgVal);
        fill(255, 255, 0);
        textSize(55);
        text("Maze Game", 200, 130);
        fill(0, 255, 255);
        textSize(20);
        text("Arrow keys/WASD to move\nUse shift to sprint\nFind the exit.", 200, 220);
        fill(255, 0, 0);
        text("(Click anywhere to Start)", 200, 280);
        
        if (transition === false) {
            if (mouseIsPressed) {
                startTransition("Play");
            }
        }
    } else if (scene === "Play") {
        if (!gameOver) {
            //Get player input and move the player
            if (player.isSprinting === false) {
                if (keyPress[SHIFT] && player.stamina > 0) {
                    player.isSprinting = true;
                    player.speed *= 2;
                    player.rotSpeed *= 2;
                }
                if (keyPress[SHIFT] === false && player.stamina < player.maxStamina) {
                    player.stamina += 3;
                }
            } else {
                if (keyPress[SHIFT] === false || player.stamina <= 0) {
                    player.isSprinting = false;
                    player.speed /= 2;
                    player.rotSpeed /= 2;
                }
            }
            
            if (keyPress[LEFT] || keyPress[65]) {
                player.angle -= player.rotSpeed;
            }
            if (keyPress[RIGHT] || keyPress[68]) {
                player.angle += player.rotSpeed;
            }
            if (player.angle < -180) {
                player.angle += 360;
            }
            if (player.angle > 180) {
                player.angle -= 360;
            }
            
            if (keyPress[UP] || keyPress[87]) {
                player.x += cos(player.angle) * player.speed;
                player.y += sin(player.angle) * player.speed;
                if (player.isSprinting) {
                    player.stamina -= 1;
                    if (millis() - player.moveTimer > 0.5 * 1000) {
                        player.moveTimer = millis();
                        playSound(getSound("retro/hit1"));
                    }
                } else {
                    if (millis() - player.moveTimer > 2 * 1000) {
                        player.moveTimer = millis();
                        playSound(getSound("retro/hit1"));
                    }
                }
            }
            if (keyPress[DOWN] || keyPress[83]) {
                player.x -= cos(player.angle) * player.speed;
                player.y -= sin(player.angle) * player.speed;
                if (player.isSprinting) {
                    player.stamina -= 1;
                    if (millis() - player.moveTimer > 0.5 * 1000) {
                        player.moveTimer = millis();
                        playSound(getSound("retro/hit1"));
                    }
                } else {
                    if (millis() - player.moveTimer > 2 * 1000) {
                        player.moveTimer = millis();
                        playSound(getSound("retro/hit1"));
                    }
                }
            }
            
            //Player + wall collisions
            for (var i = 0; i < 5; i ++) {
                for (var j = 0; j < wallsInRange.length; j ++) {
                    wallsInRange[j].playerCollide();
                }
            }
            
            //Enemy spawning
            if (enemy.active === false) {
                if (millis() - enemy.spawnTimer > max(enemy.spawnInterval, enemy.startDelay) * 1000) {
                    enemy.active = true;
                    enemy.angle = (player.angle + 180) % 360;
                    enemy.x = player.x - enemy.spawnDist * cos(enemy.angle);
                    enemy.y = player.y - enemy.spawnDist * sin(enemy.angle);
                    walls.push(new Wall(enemy.x + cos(enemy.angle + 90) * enemy.width/2, enemy.y + sin(enemy.angle + 90) * enemy.width/2, enemy.x + cos(enemy.angle - 90) * enemy.width/2, enemy.y - sin(enemy.angle + 90) * enemy.width/2, enemyTexture));
                }
            } else {
                enemy.x += cos(enemy.angle) * enemy.speed;
                enemy.y += sin(enemy.angle) * enemy.speed;
                walls[walls.length - 1].x = enemy.x + cos(enemy.angle + 90) * enemy.width/2;
                walls[walls.length - 1].y = enemy.y + sin(enemy.angle + 90) * enemy.width/2;
                walls[walls.length - 1].x2 = enemy.x + cos(enemy.angle - 90) * enemy.width/2;
                walls[walls.length - 1].y2 = enemy.y + sin(enemy.angle - 90) * enemy.width/2;
                walls[walls.length - 1].xAvg = enemy.x;
                walls[walls.length - 1].yAvg = enemy.y;
                
                enemy.dist = dist(enemy.x, enemy.y, player.x, player.y);
                
                enemy.flicker = random(0, 1) < 0.5 + enemy.dist / enemy.spawnDist / 2;
                
                //If the enemy is too close, you lose
                if (enemy.dist < enemy.width/2) {
                    gameOver = true;
                    jumpscare = true;
                    jumpscareTimer = millis();
                    playSound(getSound("retro/rumble"));
                    background(fogColour);
                    image(enemyImage, -50, -50, 500, 500);
                    jumpscareImage = get(0, 0, 400, 400);
                }
                
                if (enemy.dist > enemy.spawnDist) {
                    enemy.active = false;
                    enemy.spawnTimer = millis();
                    enemy.startDelay = 0;
                    enemy.spawnInterval = 4 + dist(player.x, player.y, winX, winY) / 1000 * 15;
                    walls.pop();
                }
            }
            
            
            //Calculate the camera plane
            cameraPos = [];
            var cameraPos1 = [player.x + cos(player.angle - viewAngle/2), player.y + sin(player.angle - viewAngle/2)];
            var cameraPos2 = [player.x + cos(player.angle + viewAngle/2), player.y + sin(player.angle + viewAngle/2)];
            for (var i = 0; i <= 1; i += 1 / resolution) {
                cameraPos.push([lerp(cameraPos1[0], cameraPos2[0], i), lerp(cameraPos1[1], cameraPos2[1], i)]);
            }
            
            
            //Generate rays for raycasting
            rays = [];
            for (var i = 0; i < cameraPos.length; i ++) {
                rays.push(new Ray(player.x, player.y, findAngle(player.x, player.y, cameraPos[i][0], cameraPos[i][1])));
            }
            
            //Find out which walls are in view/in range of the player for optimization
            wallsInRange = [];
            wallsInView = [];
            for (var i = 0; i < walls.length; i ++) {
                if (walls[i].isInRange()) {
                    wallsInRange.push(walls[i]);
                    if (walls[i].isInView()) {
                        wallsInView.push(walls[i]);
                    }
                }
            }
            
            
            background(fogColour);
            
            //Draw the beacon
            winAngle = findAngle(player.x, player.y, winX, winY);
            if (winAngle > player.angle - viewAngle/2 && winAngle < player.angle + viewAngle/2) {
                var winWidth = 50 / pow(dist(player.x, player.y, winX, winY), 1/2);
                for (var i = 0; i < 5; i ++) {
                    fill(255, 255, 0, 20);
                    rect((winAngle - player.angle+viewAngle/2) / viewAngle * 400, 100, i * 5 * winWidth, 200);
                }
            }
            
            //Cast the rays and render the walls at the same time
            for (var i = 0; i < rays.length; i ++) {
                var imageWidth = 400 / resolution;
                if (wallsInView.length > 0) {
                    var closest = rays[i].findClosestPoint();
                    var curPos = closest[0];
                    var xVal = floor(closest[1] * 100);
                    
                    var cameraDistPos = findIntersection([[player.x, player.y], [player.x + cos(player.angle), player.y + sin(player.angle)]], [[curPos[0], curPos[1]], [curPos[0] + cos(player.angle + 90), curPos[1] + sin(player.angle + 90)]]);
                    var curDist = dist(player.x, player.y, cameraDistPos[0], cameraDistPos[1]);
                    var cameraMaxY = sin(viewAngle/2) / cos(viewAngle/2) * curDist;
                    
                    
                    var imageHeight = 100 + (wallHeight / cameraMaxY * 100);
                    
                    if (xVal >= 0 && xVal < 100) {
                        image(closest[2][xVal], i * 400 / rays.length - imageWidth / 2, 200 - imageHeight/2, imageWidth, imageHeight);
                        
                        fill(fogColour, 255 - sq(viewRange - min(curDist, viewRange)) / sq(viewRange) * 255);
                        rect(i * 400 / rays.length, 200, imageWidth, ceil(imageHeight));
                    }
                }
                
                fill(fogColour, 110);
                if (enemy.active === false || enemy.active && enemy.flicker) {
                    for (var j = 1; j >= 0.7; j -= 0.1) {
                        var flashLightY = sqrt(sq(flashLightR * j) - sq(mouseX - i * 400 / rays.length));
                        if (flashLightY >= 0) {
                            rect(i * 400 / rays.length, (mouseY - flashLightY) / 2, imageWidth, mouseY - flashLightY);
                            rect(i * 400 / rays.length, (mouseY + flashLightY) / 2 + 200, imageWidth, (400 - mouseY) - flashLightY);
                        } else {
                            rect(i * 400 / rays.length, 200, imageWidth, 400);
                        }
                    }
                } else {
                    for (var j = 1; j >= 0.7; j -= 0.1) {
                        rect(i * 400 / rays.length, 200, imageWidth, 400);
                    }
                }
            }
            
            
            // Draw the minimap (optional)
            if (minimapIsDrawn) {
                pushMatrix();
                translate(200, 200);
                scale(0.2);
                fill(255, 255, 0, 100);
                arc(player.x, player.y, viewRange * 2, viewRange * 2, player.angle - viewAngle / 2, player.angle + viewAngle / 2);
                
                fill(0, 0, 255);
                ellipse(player.x, player.y, player.r * 2, player.r * 2);
                
                for (var i = 0; i < walls.length; i ++) {
                    walls[i].draw();
                }
                
                if (enemy.active) {
                    fill(255, 0, 0);
                    ellipse(enemy.x, enemy.y, 20, 20);
                }
                popMatrix();
            }
        } else {
            pushMatrix();
            translate(random(-15, 15), random(-15, 15));
            image(jumpscareImage, 0, 0);
            popMatrix();
            
            if (transition === false) {
                if (millis() - jumpscareTimer > 2 * 1000) {
                    startTransition("Play");
                }
            }
        }
    } else if (scene === "Win") {
        var bgVal = noise(millis() / 4000);
        background(0, 50 * bgVal, 100 * bgVal);
        fill(255, 255, 0);
        textSize(55);
        text("You Win!", 200, 150);
        fill(255, 0, 0);
        textSize(20);
        text("(Click to restart)", 200, 200);
        
        if (transition === false) {
            if (mouseIsPressed) {
                startTransition("Play");
            }
        }
    }
    
    //Scene transitions
    if (transition) {
        fill(0, 0, 0, transitionNum);
        rect(200, 200, 400, 400);
        if (transitionStep === 0) {
            transitionNum += (255 - transitionNum) / 4;
            if (transitionNum >= 254) {
                transitionStep = 1;
                scene = nextScene;
                if (scene === "Play") {
                    resetMaze();
                } else if (scene === "Win") {
                    playSound(getSound("retro/whistle1"));
                }
            }
        } else {
            transitionNum += (0 - transitionNum) / 16;
            if (transitionNum <= 1) {
                transition = false;
            }
        }
    }
};


//Store key presses in an array to handle multiple key presses
keyPressed = function() {
    keyPress[keyCode] = true;
};
keyReleased = function() {
    keyPress[keyCode] = false;
};