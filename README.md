npm install three @types/three @react-three/fiber @react-three/drei @react-three/postprocessing postprocessing maath uuid
// utils/math.ts
import * as THREE from 'three';

// 生成圆锥体（树）坐标
export const getTreePosition = (i: number, count: number, radius: number, height: number) => {
  const y = (i / count) * height; // 高度从 0 到 height
  const r = radius * (1 - y / height); // 半径随高度减小
  const angle = i * 1.5; // 螺旋排列
  const x = r * Math.cos(angle);
  const z = r * Math.sin(angle);
  return new THREE.Vector3(x, y - height / 2, z); // 居中
};

// 生成球体（混沌）坐标
export const getChaosPosition = (radius: number) => {
  const theta = Math.random() * Math.PI * 2;
  const phi = Math.acos(2 * Math.random() - 1);
  const r = radius * Math.cbrt(Math.random()); // 均匀分布在球体内
  return new THREE.Vector3(
    r * Math.sin(phi) * Math.cos(theta),
    r * Math.sin(phi) * Math.sin(theta),
    r * Math.cos(phi)
  );
};
import { useRef, useMemo } from 'react';
import { useFrame } from '@react-three/fiber';
import * as THREE from 'three';

export const MagicalGoldDust = ({ count = 200 }) => {
  const mesh = useRef<THREE.InstancedMesh>(null);
  const dummy = useMemo(() => new THREE.Object3D(), []);
  
  // 初始化粒子状态：位置、速度
  const particles = useMemo(() => {
    return new Array(count).fill(0).map(() => ({
      pos: new THREE.Vector3((Math.random() - 0.5) * 30, (Math.random() - 0.5) * 30, (Math.random() - 0.5) * 30),
      vel: new THREE.Vector3(0, 0, 0),
      scale: Math.random() * 0.1 + 0.05
    }));
  }, [count]);

  useFrame((state) => {
    if (!mesh.current) return;

    // 获取光标在 3D 空间的大致位置 (投影到 z=0 平面或基于相机)
    const cursorTarget = new THREE.Vector3(state.pointer.x * 15, state.pointer.y * 15, 5);

    particles.forEach((p, i) => {
      // 1. 引力逻辑
      const dir = new THREE.Vector3().copy(cursorTarget).sub(p.pos);
      const dist = dir.length();
      dir.normalize();
      
      // 距离越近，引力越大，但设置上限防止飞出
      const force = Math.min(10 / (dist * dist + 0.1), 0.5); 
      
      p.vel.add(dir.multiplyScalar(force * 0.05)); // 施加力
      p.vel.multiplyScalar(0.96); // 摩擦力
      
      // 2. 随机布朗运动 (漂浮感)
      p.vel.add(new THREE.Vector3((Math.random()-0.5)*0.01, (Math.random()-0.5)*0.01, (Math.random()-0.5)*0.01));

      p.pos.add(p.vel);

      // 更新矩阵
      dummy.position.copy(p.pos);
      dummy.scale.setScalar(p.scale);
      dummy.rotation.set(Math.sin(state.clock.elapsedTime + i), Math.cos(state.clock.elapsedTime + i), 0);
      dummy.updateMatrix();
      mesh.current!.setMatrixAt(i, dummy.matrix);
    });
    mesh.current.instanceMatrix.needsUpdate = true;
  });

  return (
    <instancedMesh ref={mesh} args={[undefined, undefined, count]}>
      <dodecahedronGeometry args={[1, 0]} />
      <meshStandardMaterial 
        color="#FFD700" 
        emissive="#FFD700" 
        emissiveIntensity={2} 
        roughness={0} 
        metalness={1} 
      />
    </instancedMesh>
  );
};
import { useMemo, useRef } from 'react';
import { useFrame } from '@react-three/fiber';
import * as THREE from 'three';
import { getChaosPosition, getTreePosition } from './utils';

const vertexShader = `
  uniform float uTime;
  uniform float uProgress; // 0 = Chaos, 1 = Tree
  attribute vec3 aTargetPos;
  attribute float aRandom;
  
  varying float vAlpha;

  // 使用 cubic-bezier 或 smoothstep 优化插值
  float easeInOut(float t) {
    return t * t * (3.0 - 2.0 * t);
  }

  void main() {
    vec3 startPos = position; // 初始位置即 chaos 位置
    vec3 endPos = aTargetPos;
    
    // 基于粒子的随机值偏移 mix 进度，让动画有层次感
    float p = smoothstep(0.0, 1.0, uProgress);
    
    vec3 currentPos = mix(startPos, endPos, p);
    
    // 添加一点风吹动的效果
    if (p > 0.8) {
       currentPos.x += sin(uTime * 2.0 + currentPos.y) * 0.1;
    }

    vec4 mvPosition = modelViewMatrix * vec4(currentPos, 1.0);
    gl_Position = projectionMatrix * mvPosition;
    gl_PointSize = (100.0 / -mvPosition.z) * (0.5 + aRandom); // 随距离缩放
    vAlpha = 0.6 + 0.4 * sin(uTime + aRandom * 10.0); // 闪烁效果
  }
`;

const fragmentShader = `
  varying float vAlpha;
  void main() {
    // 圆形粒子
    if (length(gl_PointCoord - vec2(0.5, 0.5)) > 0.5) discard;
    
    // 祖母绿基调 + 金色高光
    vec3 color = mix(vec3(0.0, 0.2, 0.1), vec3(1.0, 0.84, 0.0), 0.1); 
    gl_FragColor = vec4(color, vAlpha);
  }
`;

export const FoliageSystem = ({ isFormed }: { isFormed: boolean }) => {
  const count = 15000;
  const materialRef = useRef<THREE.ShaderMaterial>(null);
  
  const { positions, targetPositions, randoms } = useMemo(() => {
    const pos = new Float32Array(count * 3);
    const target = new Float32Array(count * 3);
    const rnd = new Float32Array(count);
    
    for (let i = 0; i < count; i++) {
      const chaos = getChaosPosition(15);
      const tree = getTreePosition(i, count, 6, 12);
      
      pos.set([chaos.x, chaos.y, chaos.z], i * 3);
      target.set([tree.x, tree.y, tree.z], i * 3);
      rnd[i] = Math.random();
    }
    return { positions: pos, targetPositions: target, randoms: rnd };
  }, []);

  useFrame((state, delta) => {
    if (materialRef.current) {
      materialRef.current.uniforms.uTime.value = state.clock.elapsedTime;
      // 动态插值 uProgress
      const target = isFormed ? 1 : 0;
      materialRef.current.uniforms.uProgress.value = THREE.MathUtils.lerp(
        materialRef.current.uniforms.uProgress.value,
        target,
        delta * 1.5 // 变形速度
      );
    }
  });

  return (
    <points>
      <bufferGeometry>
        <bufferAttribute attach="attributes-position" count={count} array={positions} itemSize={3} />
        <bufferAttribute attach="attributes-aTargetPos" count={count} array={targetPositions} itemSize={3} />
        <bufferAttribute attach="attributes-aRandom" count={count} array={randoms} itemSize={1} />
      </bufferGeometry>
      <shaderMaterial
        ref={materialRef}
        vertexShader={vertexShader}
        fragmentShader={fragmentShader}
        uniforms={{
          uTime: { value: 0 },
          uProgress: { value: 0 }
        }}
        transparent
        depthWrite={false}
        blending={THREE.AdditiveBlending}
      />
    </points>
  );
};
import { useRef, useState } from 'react';
import { useFrame, useThree } from '@react-three/fiber';
import * as THREE from 'three';

export const SwipeRotator = ({ children }: { children: React.ReactNode }) => {
  const groupRef = useRef<THREE.Group>(null);
  const { size, viewport } = useThree();
  
  const [isDragging, setIsDragging] = useState(false);
  const velocity = useRef(0);
  const lastX = useRef(0);

  const handlePointerDown = (e: any) => {
    setIsDragging(true);
    lastX.current = e.clientX;
    velocity.current = 0;
  };

  const handlePointerUp = () => {
    setIsDragging(false);
  };

  const handlePointerMove = (e: any) => {
    if (isDragging) {
      const deltaX = e.clientX - lastX.current;
      lastX.current = e.clientX;
      // 计算这一帧的速度，灵敏度调整
      velocity.current = deltaX * 0.005; 
      if (groupRef.current) {
        groupRef.current.rotation.y += velocity.current;
      }
    }
  };

  useFrame(() => {
    if (!isDragging && groupRef.current) {
      // 惯性与摩擦力模拟
      groupRef.current.rotation.y += velocity.current;
      velocity.current *= 0.95; // 摩擦系数，越接近 1 滑得越久
      
      // 防止无限微小旋转
      if (Math.abs(velocity.current) < 0.0001) velocity.current = 0;
    }
  });

  return (
    <group 
      ref={groupRef} 
      onPointerDown={handlePointerDown} 
      onPointerUp={handlePointerUp} 
      onPointerLeave={handlePointerUp}
      onPointerMove={handlePointerMove}
    >
      {children}
    </group>
  );
};
import React, { useState, Suspense } from 'react';
import { Canvas } from '@react-three/fiber';
import { Environment, OrbitControls, Loader } from '@react-three/drei';
import { EffectComposer, Bloom } from '@react-three/postprocessing';
import { SwipeRotator } from './SwipeRotator';
import { FoliageSystem } from './FoliageSystem';
import { MagicalGoldDust } from './MagicalGoldDust';

// 装饰品组件 (简化版：使用 InstanceMesh 渲染球体)
const Ornaments = ({ isFormed }: { isFormed: boolean }) => {
    // 此处逻辑与 FoliageSystem 类似，但使用 useFrame 驱动 InstancedMesh 的矩阵
    // 为了代码简洁，这里略去具体实现，核心逻辑同样是 ChaosPos -> TargetPos 的 Lerp
    // 材质设为高光金色 MeshStandardMaterial
    return null; // 占位
}

const SceneContent = ({ isFormed }: { isFormed: boolean }) => {
  return (
    <>
      <ambientLight intensity={0.5} color="#002419" />
      <pointLight position={[10, 10, 10]} intensity={1.5} color="#FFD700" />
      <spotLight position={[-10, 20, 10]} angle={0.3} penumbra={1} intensity={2} color="#ffffff" castShadow />

      {/* 挥动交互层 */}
      <SwipeRotator>
         <FoliageSystem isFormed={isFormed} />
         {/* <Ornaments isFormed={isFormed} /> */}
      </SwipeRotator>

      {/* 魔法金粉 (不受旋转影响，或根据需求放入 Rotator) */}
      <MagicalGoldDust count={300} />

      {/* 环境光贴图 - 营造大堂感 */}
      <Environment preset="lobby" background={false} />

      {/* 后期处理 */}
      <EffectComposer disableNormalPass>
        <Bloom 
            luminanceThreshold={0.8} 
            mipmapBlur 
            intensity={1.2} 
            radius={0.6}
        />
      </EffectComposer>
    </>
  );
};

export default function App() {
  const [isFormed, setIsFormed] = useState(false);

  return (
    <div className="w-full h-screen bg-[#00100a] text-[#FFD700] font-serif overflow-hidden select-none">
      
      {/* 3D 画布 */}
      <Canvas
        shadows
        camera={{ position: [0, 4, 20], fov: 45 }}
        gl={{ antialias: false, toneMappingExposure: 1.5 }} // 关闭 AA 交给后期，提高曝光
        dpr={[1, 2]}
      >
        <Suspense fallback={null}>
            <SceneContent isFormed={isFormed} />
        </Suspense>
      </Canvas>

      {/* UI 覆盖层 */}
      <div className="absolute top-0 left-0 w-full p-8 pointer-events-none flex justify-between items-start z-10">
        <div>
          <h1 className="text-4xl md:text-6xl font-bold tracking-widest drop-shadow-[0_0_10px_rgba(255,215,0,0.5)]">
            GRAND LUXURY
          </h1>
          <p className="text-xl mt-2 italic text-emerald-300">Interactive Christmas Tree</p>
        </div>
      </div>

      <div className="absolute bottom-10 left-1/2 transform -translate-x-1/2 z-10">
        <button
          onClick={() => setIsFormed(!isFormed)}
          className="pointer-events-auto px-8 py-3 border-2 border-[#FFD700] text-[#FFD700] 
                     bg-black/30 backdrop-blur-md rounded-full text-xl font-bold uppercase tracking-wider
                     hover:bg-[#FFD700] hover:text-black transition-all duration-500
                     shadow-[0_0_20px_rgba(255,215,0,0.3)]"
        >
          {isFormed ? "Release Chaos" : "Assemble Tree"}
        </button>
      </div>

      <Loader />
    </div>
  );
}
