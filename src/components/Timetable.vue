<script setup lang="ts">
import { ref, computed } from 'vue';

/**
 * 课程类型定义
 * @property {number} id - 课程唯一标识符
 * @property {string} name - 课程名称
 * @property {number} size - 课程宽度，1-5个格子
 * @property {number} row - 课程所在行，0-1
 * @property {number} startCol - 课程起始列，0-4
 */
interface Course {
  id: number;
  name: string;
  size: number; // 1-5
  row: number; // 0-1
  startCol: number; // 0-4
}

// 课程表数据
const courses = ref<Course[]>([]);
const nextId = ref(1);

// 拖拽相关
const draggedCourse = ref<Course | null>(null);
const dragTarget = ref<{ row: number; col: number } | null>(null);

/**
 * 开始拖拽课程
 * @param {Course} course - 要拖拽的课程
 */
const dragStart = (course: Course) => {
  // 当正在调整尺寸时，不触发拖动事件
  if (resizingCourse.value) return;
  draggedCourse.value = course;
};

/**
 * 拖拽经过格子时的处理
 * @param {DragEvent} event - 拖拽事件
 * @param {number} row - 格子所在行
 * @param {number} col - 格子所在列
 */
const dragOver = (event: DragEvent, row: number, col: number) => {
  event.preventDefault();
  // 更新拖动目标位置
  dragTarget.value = { row, col };
};

/**
 * 拖拽离开格子时的处理
 */
const dragLeave = () => {
  // 离开时清除拖动目标位置
  dragTarget.value = null;
};

// 调整尺寸相关
const resizingCourse = ref<Course | null>(null);
const resizeStartX = ref(0);
const resizeStartSize = ref(0);
const resizeStartCol = ref(0);

/**
 * 开始调整课程尺寸
 * @param {Course} course - 要调整尺寸的课程
 * @param {MouseEvent} event - 鼠标事件
 * @param {'left' | 'right'} direction - 调整方向，默认为右侧
 */
const startResize = (course: Course, event: MouseEvent, direction: 'left' | 'right' = 'right') => {
  // 阻止事件冒泡，避免触发dragstart
  event.stopPropagation();
  // 阻止默认行为
  event.preventDefault();
  
  resizingCourse.value = course;
  resizeStartX.value = event.clientX;
  resizeStartSize.value = course.size;
  resizeStartCol.value = course.startCol;
  
  // 使用箭头函数确保this指向正确
  const handleMouseMove = (e: MouseEvent) => {
    if (!resizingCourse.value) return;
    
    const deltaX = e.clientX - resizeStartX.value;
    const cellWidth = 100; // 假设每个格子的宽度为100px
    const deltaSize = Math.round(deltaX / cellWidth);
    
    if (direction === 'right') {
      const newSize = Math.max(1, Math.min(5, resizeStartSize.value + deltaSize));
      
      // 检查新尺寸是否可用
      if (canResizeCourse(resizingCourse.value, newSize)) {
        resizingCourse.value.size = newSize;
      }
    } else {
      // 左侧边缘调整
      const deltaCol = deltaSize;
      const newStartCol = Math.max(0, Math.min(4, resizeStartCol.value + deltaCol));
      const newSize = Math.max(1, Math.min(5, resizeStartSize.value - deltaSize));
      
      // 检查新位置和尺寸是否可用
      if (newStartCol + newSize <= 5) {
        let canResize = true;
        for (let col = newStartCol; col < newStartCol + newSize; col++) {
          // 检查是否有其他课程占用这个位置
          let isOccupied = false;
          for (const other of courses.value) {
            if (other && other !== resizingCourse.value && other.row === resizingCourse.value.row) {
              if (col >= other.startCol && col < other.startCol + other.size) {
                isOccupied = true;
                break;
              }
            }
          }
          if (isOccupied) {
            canResize = false;
            break;
          }
        }
        
        if (canResize) {
          resizingCourse.value.startCol = newStartCol;
          resizingCourse.value.size = newSize;
        }
      }
    }
  };
  
  const handleMouseUp = () => {
    resizingCourse.value = null;
    document.removeEventListener('mousemove', handleMouseMove);
    document.removeEventListener('mouseup', handleMouseUp);
  };
  
  // 绑定事件监听器
  document.addEventListener('mousemove', handleMouseMove);
  document.addEventListener('mouseup', handleMouseUp);
};

/**
 * 计算每个格子的状态
 * @returns {boolean[][]} 2x5的网格状态，true表示被占用，false表示空闲
 */
const gridStatus = computed(() => {
  const status = Array(2).fill(null).map(() => Array(5).fill(false));
  
  courses.value.forEach(course => {
    if (course && course !== draggedCourse.value && course !== resizingCourse.value) {
      for (let col = course.startCol; col < course.startCol + course.size; col++) {
        status[course.row][col] = true;
      }
    }
  });
  
  return status;
});

/**
 * 检查位置是否可用
 * @param {number} row - 行号
 * @param {number} startCol - 起始列号
 * @param {number} size - 课程宽度
 * @returns {boolean} 是否可用
 */
const isPositionAvailable = (row: number, startCol: number, size: number): boolean => {
  if (startCol + size > 5) return false;
  
  for (let col = startCol; col < startCol + size; col++) {
    if (gridStatus.value[row][col]) {
      return false;
    }
  }
  return true;
};

/**
 * 检查课程是否可以调整到新尺寸
 * @param {Course} course - 要调整的课程
 * @param {number} newSize - 新尺寸
 * @returns {boolean} 是否可以调整
 */
const canResizeCourse = (course: Course, newSize: number): boolean => {
  if (newSize < 1 || newSize > 5) return false;
  if (course.startCol + newSize > 5) return false;
  
  for (let col = course.startCol; col < course.startCol + newSize; col++) {
    // 检查是否有其他课程占用这个位置
    let isOccupied = false;
    for (const other of courses.value) {
      if (other && other !== course && other.row === course.row) {
        if (col >= other.startCol && col < other.startCol + other.size) {
          isOccupied = true;
          break;
        }
      }
    }
    if (isOccupied) {
      return false;
    }
  }
  return true;
};

/**
 * 添加课程
 * @param {number} row - 行号
 * @param {number} col - 列号
 */
const addCourse = (row: number, col: number) => {
  if (gridStatus.value[row][col]) {
    alert('该位置已被占用');
    return;
  }
  
  // 计算最大可用尺寸
  let maxSize = 5 - col;
  for (let size = 1; size <= maxSize; size++) {
    if (!isPositionAvailable(row, col, size)) {
      maxSize = size - 1;
      break;
    }
  }
  
  if (maxSize === 0) {
    alert('该位置无法添加课程');
    return;
  }
  
  // 提示用户选择课程大小
  const sizeInput = prompt(`请输入课程大小 (1-${maxSize})`, '1');
  const size = parseInt(sizeInput || '1');
  
  if (isNaN(size) || size < 1 || size > maxSize) {
    alert(`无效的课程大小，请输入 1-${maxSize} 之间的数字`);
    return;
  }
  
  const name = prompt('请输入课程名称', '新课程');
  if (!name) return;
  
  courses.value.push({
    id: nextId.value++,
    name,
    size,
    row,
    startCol: col
  });
};

/**
 * 获取目标位置上的课程
 * @param {number} row - 行号
 * @param {number} col - 列号
 * @returns {Course | null} 目标位置上的课程，没有则返回null
 */
const getCourseAtPosition = (row: number, col: number): Course | null => {
  for (const course of courses.value) {
    if (course && course !== draggedCourse.value && course.row === row) {
      if (col >= course.startCol && col < course.startCol + course.size) {
        return course;
      }
    }
  }
  return null;
};

/**
 * 检查两个课程的位置是否重叠
 * @param {Course} course1 - 第一个课程
 * @param {Course} course2 - 第二个课程
 * @returns {boolean} 是否重叠
 */
const hasOverlap = (course1: Course, course2: Course): boolean => {
  if (course1.row !== course2.row) return false;
  const start1 = course1.startCol;
  const end1 = course1.startCol + course1.size;
  const start2 = course2.startCol;
  const end2 = course2.startCol + course2.size;
  return start1 < end2 && start2 < end1;
};

/**
 * 检查课程是否可以放置在某个位置
 * @param {Course} course - 要放置的课程
 * @param {number} row - 行号
 * @param {number} startCol - 起始列号
 * @param {Course | null} excludeCourse - 排除的课程（用于互换位置时）
 * @returns {boolean} 是否可以放置
 */
const canPlaceCourse = (course: Course, row: number, startCol: number, excludeCourse: Course | null): boolean => {
  if (startCol < 0 || startCol + course.size > 5) return false;

  const tempCourse: Course = { ...course, row, startCol };
  for (const other of courses.value) {
    if (other && other !== course && other !== excludeCourse) {
      if (hasOverlap(tempCourse, other)) {
        return false;
      }
    }
  }
  return true;
};

/**
 * 检查某个位置是否可以放置当前拖动的课程
 * @param {number} row - 行号
 * @param {number} col - 列号
 * @returns {boolean} 是否可以放置
 */
const isDroppable = (row: number, col: number): boolean => {
  if (!draggedCourse.value) return false;
  return isPositionAvailable(row, col, draggedCourse.value.size);
};

/**
 * 放置课程
 * @param {number} row - 行号
 * @param {number} col - 列号
 */
const drop = (row: number, col: number) => {
  if (!draggedCourse.value) return;

  const dragged = draggedCourse.value;

  // 检查目标位置是否有其他课程
  const targetCourse = getCourseAtPosition(row, col);

  if (targetCourse) {
    // 尝试互换位置
    const draggedNewRow = targetCourse.row;
    const draggedNewCol = targetCourse.startCol;
    const targetNewRow = dragged.row;
    const targetNewCol = dragged.startCol;

    // 检查互换后两个课程是否都能放下
    const canDraggedMove = canPlaceCourse(dragged, draggedNewRow, draggedNewCol, targetCourse);
    const canTargetMove = canPlaceCourse(targetCourse, targetNewRow, targetNewCol, dragged);

    if (canDraggedMove && canTargetMove) {
      // 互换位置
      const tempRow = dragged.row;
      const tempCol = dragged.startCol;
      dragged.row = draggedNewRow;
      dragged.startCol = draggedNewCol;
      targetCourse.row = tempRow;
      targetCourse.startCol = tempCol;

      // 检查调换后是否存在重叠
      if (hasOverlap(dragged, targetCourse)) {
        // 找出尺寸较小的课程
        const smallerCourse = dragged.size <= targetCourse.size ? dragged : targetCourse;
        const largerCourse = dragged.size > targetCourse.size ? dragged : targetCourse;

        // 计算重叠部分，将较小的课程往右推
        let newStartCol = largerCourse.startCol + largerCourse.size;
        // 确保新位置不会超出课程表范围
        if (newStartCol + smallerCourse.size <= 5) {
          smallerCourse.startCol = newStartCol;
        }
      }
    } else {
      alert('这两个课程无法互换位置');
    }
  } else {
    // 检查新位置是否可用（需要连续的size个格子）
    if (!isPositionAvailable(row, col, dragged.size)) {
      alert('该位置无法放置课程，需要连续的' + dragged.size + '个格子');
      return;
    }

    // 更新课程位置
    dragged.row = row;
    dragged.startCol = col;
  }

  draggedCourse.value = null;
};

/**
 * 移除课程
 * @param {Course} course - 要移除的课程
 */
const removeCourse = (course: Course) => {
  courses.value = courses.value.filter(c => c.id !== course.id);
};
</script>

<template>
  <div class="timetable-container">
    <h2>课程表</h2>
    <div class="timetable">
      <div 
        v-for="(_, rowIndex) in 2" 
        :key="rowIndex" 
        class="timetable-row"
      >
        <!-- 渲染格子背景 -->
        <div 
          v-for="(_, colIndex) in 5" 
          :key="colIndex"
          class="timetable-cell"
          :class="{ 
            occupied: gridStatus[rowIndex][colIndex],
            droppable: isDroppable(rowIndex, colIndex)
          }"
          @click="!gridStatus[rowIndex][colIndex] && addCourse(rowIndex, colIndex)"
          @dragover="dragOver($event, rowIndex, colIndex)"
          @dragleave="dragLeave"
          @drop="drop(rowIndex, colIndex)"
        ></div>
        
        <!-- 渲染课程 -->
        <template v-for="course in courses" :key="course.id">
          <div 
            v-if="course.row === rowIndex"
            class="course"
            :style="{
              left: `calc(${course.startCol} * ((100% - 40px) / 5 + 10px))`,
              width: `calc((${course.size} * (100% - 40px)) / 5 + ${(course.size - 1) * 10}px)`
            }"
            draggable="true"
            @dragstart="dragStart(course)"
            @dragover="dragOver($event, course.row, course.startCol)"
            @drop="drop(course.row, course.startCol)"
          >
            <div class="resize-handle left" @mousedown="startResize(course, $event, 'left')"></div>
            <div class="course-content">
              <span>{{ course.name }}</span>
              <button class="remove-btn" @click.stop="removeCourse(course)">×</button>
            </div>
            <div class="resize-handle right" @mousedown="startResize(course, $event, 'right')"></div>
          </div>
        </template>
      </div>
    </div>
    <div class="instructions">
      <p>点击空白格子添加课程</p>
      <p>拖拽课程调整位置</p>
      <p>点击课程上的 × 移除课程</p>
    </div>
  </div>
</template>

<style scoped>
.timetable-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

h2 {
  text-align: center;
  color: #42b983;
}

.timetable {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin: 20px 0;
}

.timetable-row {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 10px;
  position: relative;
  margin-bottom: 10px;
}

.timetable-cell {
  width: 100%;
  height: 100px;
  border: 2px solid #e0e0e0;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s ease;
}

.timetable-cell:hover:not(.occupied) {
  border-color: #42b983;
  background-color: #f0f9f4;
}

.timetable-cell.droppable {
  border-color: #42b983;
  background-color: #d4f1e3;
  cursor: move;
}

.timetable-cell.occupied {
  cursor: default;
}

.course {
  position: absolute;
  top: 0;
  height: 100px;
  background-color: #42b983;
  color: white;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: move;
  z-index: 10;
  transition: all 0.2s ease;
  box-sizing: border-box;
}

.course:hover {
  background-color: #35495e;
}

.course-content {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 90%;
  padding: 0 10px;
}

.remove-btn {
  background: none;
  border: none;
  color: white;
  font-size: 18px;
  cursor: pointer;
  padding: 0;
  width: 20px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  transition: all 0.2s ease;
}

.remove-btn:hover {
  background-color: rgba(255, 255, 255, 0.2);
}

.resize-handle {
  position: absolute;
  top: 0;
  bottom: 0;
  width: 10px;
  background-color: rgba(255, 255, 255, 0.3);
  cursor: ew-resize;
  transition: background-color 0.2s ease;
}

.resize-handle.left {
  left: 0;
}

.resize-handle.right {
  right: 0;
}

.resize-handle:hover {
  background-color: rgba(255, 255, 255, 0.5);
}

.instructions {
  margin-top: 30px;
  padding: 15px;
  background-color: #f8f9fa;
  border-radius: 4px;
}

.instructions p {
  margin: 5px 0;
  color: #666;
}
</style>