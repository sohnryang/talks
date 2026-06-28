# Reading Group Outline

## Motivation

- Auto-vectorization에는 크게 두 가지 접근이 있음: loop-vectorization vs. SLP
- Loop vectorization: loop body에 있는 scalar instruction을 widen
  - CRAY와 같은 당시의 long-vector architecture에 영향을 받음
  - 복잡한 loop dependence analysis가 필요
- SLP: straight-line code에서 isomorphic한 instruction 여러 개를 packing
  - Multimedia extension에 있는 short-vector architecture의 영향을 받음
  - Dependence analysis가 간단해지지만, control flow가 있으면 적용하기 어려워짐
- LV와 SLP는 최적화를 잘 하는 코드의 종류가 다르기 때문에 gcc, LLVM과 같은 대부분의 컴파일러는 둘을 모두 구현
- SLP를 generalize해서 loop를 다루게 만들 수 있을까?

## Extending SLP

- SLP를 generalize해서 loop를 다루게 만들려면 control flow를 다룰 수 있어야 함
- SLP로 loop-level 최적화를 하려면 code motion을 할 수 있어야 함
  - 이 경우 basic block 밖으로 instruction을 움직여야 하는데, SLP는 straight-line code를 지원하도록 설계되기 때문에 원래 어려움
- if-conversion도 해야함
- Predicated SSA를 활용, code motion을 단순히 instruction reordering으로써 수행하도록 변환
  - CFG를 Predicated SSA로 바꾸고, SLP를 수행한 다음 다시 CFG로 변환

## Predicated SSA

- CFG를 전부 straight-line code로 변환
- 각 instruction에 control predicate이 붙어 있어서, control predicate이 true일 때만 실행
- PHI node를 분류, loop header에 있는 것은 $\mu$-node로 바꿈

### Conversion to Predicated SSA

- 우선 loop가 canonical form이어야 함
  - single incoming edge
  - single back edge
  - dedicated loop header
  - dedicated loop preheader
  - dedicated loop latch
- forward PHI node를 gated PHI node로 바꾸고, header PHI를 mu node로 변경
- control predicate를 구하기 위해 symbolic execution을 할 수도 있겠지만 조건이 너무 복잡하게 나옴
  - 따로 simplification을 해야 하는데 이것이 힘듬
- 어떤 basic block $b$가 실행되려면 두 조건이 만족해야 함
  - $b$가 control-dependent한 block $b'$이 실행되고
    - $b$의 post dominance frontier를 계산, control-depndent한 block을 얻을 수 있음
  - $b'$에서 $b$로 branch가 있어야 함
    - $b'$은 $b$의 post dominance frontier에 있으므로, $b' \to b''$ branch가 있어서 $b''$은 $b'$의 successor이고, $b$는 $b''$을 postdominate함
    - 이런 $b''$은 unique (일반적으로는 여러 개 존재할 수도 있지만 여기서는 switch와 같은 branch는 허용하지 않으므로 successor가 at most 2개라서 성립)
- 같은 basic block에 있는 instruction들은 같은 control predicate을 가짐
  - block마다 control predicate을 계산
  - control-flow equivalence: loop header(혹은 entry block)와 control flow equivalent하면 `true` control predicate
    - control-flow equivalence는 dom/postdom을 이용한 휴리스틱으로 계산
  - 그렇지 않을 경우, control dependent한 block의 successor 중 postdom한 것의 branch를 타는 control predicate의 disjuction으로 계산

## Vector packing

- Predicated SSA로 나타냈다면 이제 SLP를 돌려야 함
- Packing heuristic과 상관없이 vectorization 할 수 있음
- 논문에서는 간단한 bottom-up heuristic 사용
  - Vectorizable seed instruction을 모음 (연속된 메모리 주소에 store, reduction 등)
  - use-def chain을 따라가면서 nonvectorizable instruction을 방문할 때까지 vectorizable instruction을 모음
  - 탐색이 끝났을 때 dependency cycle이 없는지 확인하고, cost를 확인해 packing decision
  - Packing할 instruction이 다른 loop에 포함되는 경우에는 같은 loop에 포함되도록 transform
    - Loop fusion (independent하고 trip count, control predicate가 같음) 혹은 coiteration (control predicate 혹은 trip count가 다름) 수행
  - Scheduling 뒤 vector instruction을 emit하고, control predicate 할당
    - PHI node, load/store pack의 경우는 추가적으로 처리
      - PHI의 경우는 필요에 따라 select를 emit하고, load/store는 mask emit
    - control predicate은 packing된 instruction의 control predicate에서 imply되는 조건 중 가장 강한 것을 고름

## Conversion to CFG

- PHI를 모두 move로 바꾸고, recursive하게 loop lowering한 뒤 다시 fixup
- control predicate의 disjuction을 만났을 때에는 sub-term에 해당하는 basic block을 쿼리한 다음, block들을 join하도록 emit
- conjunction은 $p \land c$ where $c$ is condition일 때, $p$에 해당하는 basic block에서 $c$에 해당하는 conditional branch를 새로 만든 block으로 향하게 하면 됨

## Implementation details

- Loop unrolling: loop level parallelism을 드러내기 위해 수행
  - Top down으로 parent loop부터 traverse
  - 가상으로 한번 unroll 해보고 SLP packing이 가능한지 확인
- Dependence analysis: loop-carried depndence를 볼 필요는 없지만 loop fusion vs coiteration 결정을 위해서는 loop 사이의 dependence를 볼 필요가 있음
  - use-def dependence와 memory dependence 확인
  - SCEV, AA 사용
- Cost model
  - Scalar instruction을 vector instruction으로 대체해야 성능 향상이 생김
  - Function call 등 nonvectorizable instruction 여러 개를 실행해 shuffle하는 경우 shuffle cost까지 고려

## Evaluation

- TSVC, PolyBench, ISPC에서 좋은 결과를 얻음
- Nested loop에서 outer loop를 unroll, loop guard 사이의 instruction을 packing해 성능 향상을 얻기도 했음