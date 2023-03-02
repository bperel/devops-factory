<script setup>
import { computed, onMounted, ref, watch } from "vue";

import {
  Engine,
  Render,
  World,
  Bodies,
  Body,
  Events,
  Composite,
  Composites,
  Query,
  Constraint,
  Mouse,
  MouseConstraint,
} from "matter-js";
import Options from "./Options.vue";
import PulumiForm from "./PulumiForm.vue";
import decomp from "poly-decomp";

const bps = ref(0);
const playbackSpeed = ref(1);
const boxes = ref([]);
const json = ref(
  JSON.stringify({
    cpu: 256,
    memory: 512,
  })
);

let slope = null;
const bpsInterval = ref(null);

const cpu = computed(() => json.value.cpu);
const friction = computed(() => 1 / cpu.value);
const memory = computed(() => json.value.memory);
const slopePosition = computed(() =>
  Math.min(600, 270 + (600 - 270) * (memory.value / 1024))
);
const rayEnd = computed(() => slopePosition.value - 125);

window.decomp = decomp;
const width = window.innerWidth,
  height = window.innerHeight - 55,
  engineOptions = {
    enableSleeping: true,
    timing: {
      timeScale: 0.5,
    },
  },
  renderOptions = {
    width,
    height,
    wireframes: false,
    wireframeBackground: "transparent",
    background: "transparent",
    //background: "#ffffff99",
    showSleeping: false,
    showDebug: false,
    showBroadphase: false,
    showBounds: false,
    showVelocity: false,
    showCollisions: false,
    showSeparations: false,
    showAxes: false,
    showPositions: false,
    showAngleIndicator: false,
    showIds: false,
    showShadows: false,
    showVertexNumbers: false,
    showConvexHulls: false,
    showInternalEdges: false,
    showMousePosition: true,
  };
// create an engine
const engine = Engine.create(engineOptions),
  world = engine.world;

const addRope = (x, y) => {
  const group = Body.nextGroup(true);

  const rope = Composites.stack(x, y, 10, 1, 10, 0, (x, y) =>
    Bodies.rectangle(x - 20, y, 50, 20, {
      collisionFilter: { group: group },
      chamfer: 5,
      render: {
        fillStyle: "grey",
      },
    })
  );

  Composites.chain(rope, 0.3, 0, -0.3, 0, { stiffness: 1, length: 0 });

  Composite.add(world, rope);

  return rope;
};

const addBox = (boxSide) => {
  const box = Bodies.rectangle(250, 0, boxSide, boxSide, {
    render: {
      label: "request",
      fillStyle: "red",
      sprite: {
        texture: "box.png",
      },
    },
    friction: friction.value,
  });
  boxes.value.push(box);
  Composite.add(world, box);
};

const createConveyorBelt = () => {
  const scale = 0.9;
  const [xx, yy, width, height, wheelSize] = [
    350,
    570,
    150 * scale,
    30 * scale,
    30 * scale,
  ];
  const group = Body.nextGroup(true),
    wheelBase = 20,
    wheelAOffset = -width * 0.5 + wheelBase,
    wheelBOffset = width * 0.5 - wheelBase,
    wheelYOffset = 0;

  const car = Composite.create({ label: "Car" }),
    body = Bodies.rectangle(xx, yy, width, height, {
      collisionFilter: {
        group: group,
      },
      chamfer: {
        radius: height * 0.5,
      },
      density: 0.0002,
    });

  const wheelOptions = {
    render: {
      fillStyle: "green",
    },
    collisionFilter: {
      group: group,
    },
    friction: 0.8,
  };

  const wheelA = Bodies.circle(
    xx + wheelAOffset,
    yy + wheelYOffset,
    wheelSize,
    wheelOptions
  );

  const wheelB = Bodies.circle(
    xx + wheelBOffset,
    yy + wheelYOffset,
    wheelSize,
    wheelOptions
  );

  const axelA = Constraint.create({
    bodyB: body,
    pointB: { x: wheelAOffset, y: wheelYOffset },
    bodyA: wheelA,
    stiffness: 1,
    length: 0,
  });

  const axelB = Constraint.create({
    bodyB: body,
    pointB: { x: wheelBOffset, y: wheelYOffset },
    bodyA: wheelB,
    stiffness: 1,
    length: 0,
  });

  Composite.add(world, car);
  Composite.add(car, [body, wheelA, wheelB, axelA, axelB]);

  return car;
};

onMounted(() => {
  // create a renderer
  const render = Render.create({
    element: document.querySelector("#factory"),
    engine,
    options: renderOptions,
  });
  //Bootstrap Mouse Controls
  // add mouse control
  const mouse = Mouse.create(render.canvas),
    mouseConstraint = MouseConstraint.create(engine, {
      mouse: mouse,
      constraint: {
        stiffness: 0.5,
        render: {
          visible: true,
        },
      },
    });
  World.add(world, mouseConstraint);
  // keep the mouse in sync with rendering
  render.mouse = mouse;
  // an example of using mouse events on a mouse
  let fsa = [];
  Events.on(mouseConstraint, "startdrag", (event) => {
    fsa = event.body.parts.map((v) => v.render.fillStyle);
    event.body.parts = event.body.parts.map((v) => {
      v.render.fillStyle = "#FF0000";
      return v;
    });
  });
  Events.on(mouseConstraint, "enddrag", (event) => {
    event.body.parts.map(
      (v, i) => (event.body.parts[i].render.fillStyle = fsa[i])
    );
  });

  const wallWidth = window.innerWidth / 2,
    wallHeight = window.innerHeight * 2;
  Composite.add(world, [
    // walls
    Bodies.rectangle(wallWidth, 285, wallWidth * 1.4, 50, {
      isStatic: true,
      angle: Math.PI * 0.1,
    }),
    Bodies.rectangle(0, wallHeight * 0.4, wallWidth * 2, 50, {
      isStatic: true,
    }),
    Bodies.rectangle(150, 100, 50, wallHeight, {
      isStatic: true,
    }),
  ]);

  // Composite.add(
  //   world,
  //   Constraint.create({
  //     bodyB: rope1.bodies[rope1.bodies.length - 1],
  //     pointB: { x: -10, y: 0 },
  //     bodyA: rope2.bodies[0],
  //     pointA: { x: 10, y: 0 },
  //     stiffness: 1,
  //     length: 0,
  //   })
  // );
  // Composite.add(
  //   world,
  //   Constraint.create({
  //     bodyB: rope2.bodies[rope2.bodies.length - 1],
  //     pointB: { x: -10, y: 0 },
  //     bodyA: rope1.bodies[0],
  //     pointA: { x: 10, y: 0 },
  //     stiffness: 1,
  //     length: 0,
  //   })
  // );

  slope = Bodies.rectangle(750, 450, 1200, 20, {
    isStatic: true,
    angle: Math.PI * 0.1,
    render: { fillStyle: "#aaa", label: "slope" },
  });
  Composite.add(world, slope);

  const rope1 = addRope(200, wallHeight * 0.28);
  const rope2 = addRope(200, wallHeight * 0.35);
  createConveyorBelt();

  const startPoint = { x: 400, y: 150 };

  let collisions = [];
  Events.on(engine, "afterUpdate", () => {
    const endPoint = { x: 400, y: rayEnd.value };
    collisions = Query.ray(boxes.value, startPoint, endPoint);
    if (collisions.length > 0) {
      context.strokeStyle = "#fff";
    } else {
      context.strokeStyle = "#555";
    }
  });

  Engine.run(engine);
  Render.run(render);

  const context = render.context;

  Events.on(render, "afterRender", () => {
    const endPoint = { x: 400, y: rayEnd.value };
    Render.startViewTransform(render);
    context.beginPath();
    context.moveTo(startPoint.x, startPoint.y);
    context.lineTo(endPoint.x, endPoint.y);
    context.strokeStyle = "red";

    if (collisions.length > 0) {
      context.lineWidth = 2;
    } else {
      context.lineWidth = 0.5;
    }

    for (const collision of collisions) {
      collision.bodyA.render.sprite.texture = "firstvet.png";
    }
    context.fillStyle = "rgba(255,165,0,0.7)";
    context.fill();

    context.stroke();
    Render.endViewTransform(render);
  });
  watch(
    () => bps.value,
    (newValue) => {
      if (bpsInterval.value) {
        clearInterval(bpsInterval.value);
      }
      if (newValue) {
        bpsInterval.value = setInterval(() => {
          addBox(64);
        }, 1000 / newValue);
      }
    },
    { immediate: true }
  );

  watch(
    () => friction.value,
    (newFriction) => {
      Composite.allBodies(engine.world).forEach((box) => {
        box.friction = newFriction;
      });
    }
  );

  watch(
    () => slopePosition.value,
    (newSlopePosition) => {
      if (newSlopePosition) {
        console.log(newSlopePosition);
        Body.setPosition(slope, { x: slope.position.x, y: newSlopePosition });
      }
    },
    { immediate: true }
  );

  watch(
    () => playbackSpeed.value,
    (newPlaybackSpeed) => {
      engine.timing.timeScale = newPlaybackSpeed;
    }
  );
});
</script>

<template>
  <div id="main">
    <div id="factory"></div>
    <PulumiForm @change="json = $event" />
  </div>
  <Options
    @change-bps="bps = $event"
    @change-playback-speed="playbackSpeed = $event"
  />
</template>

<style scoped lang="scss">
#main {
  display: flex;
  align-content: start;
  width: 100vw;
  height: 100vh;
  > * {
    flex-grow: 1;
    border: 2px solid green;

    &:first-child {
      width: 70%;
    }
  }
}
</style>
