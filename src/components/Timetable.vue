<script setup lang="ts">
import { ref, computed } from 'vue';

/**
 * 课程表组件
 * 支持2x5的网格布局，可添加、拖拽、调整尺寸的课程
 * 课程尺寸支持：1×1、1×2、1×3、1×4、1×5、2×1、2×2、2×3、2×4、2×5
 */

/**
 * 课程类型定义
 * @property {number} id - 课程唯一标识符
 * @property {string} name - 课程名称
 * @property {number} size - 课程宽度，1-5个格子
 * @property {number} height - 课程高度，1-2行
 * @property {number} row - 课程所在行，0-1
 * @property {number} startCol - 课程起始列，0-4
 */
interface Course {
  id: number;
  name: string;
  size: number; // 1-5
  height: number; // 1-2
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
const resizeStartY = ref(0);
const resizeStartSize = ref(0);
const resizeStartHeight = ref(0);
const resizeStartCol = ref(0);
const resizeStartRow = ref(0);

/**
 * 开始调整课程尺寸
 * @param {Course} course - 要调整尺寸的课程
 * @param {MouseEvent} event - 鼠标事件
 * @param {'left' | 'right' | 'top' | 'bottom' | 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right'} direction - 调整方向
 */
const startResize = (course: Course, event: MouseEvent, direction: 'left' | 'right' | 'top' | 'bottom' | 'top-left' | 'top-right' | 'bottom-left' | 'bottom-right' = 'right') => {
  // 阻止事件冒泡，避免触发dragstart
  event.stopPropagation();
  // 阻止默认行为
  event.preventDefault();
  
  resizingCourse.value = course;
  resizeStartX.value = event.clientX;
  resizeStartY.value = event.clientY;
  resizeStartSize.value = course.size;
  resizeStartHeight.value = course.height;
  resizeStartCol.value = course.startCol;
  resizeStartRow.value = course.row;
  
  // 使用箭头函数确保this指向正确
  const handleMouseMove = (e: MouseEvent) => {
    if (!resizingCourse.value) return;
    
    const deltaX = e.clientX - resizeStartX.value;
    const deltaY = e.clientY - resizeStartY.value;
    const cellWidth = 100; // 假设每个格子的宽度为100px
    const cellHeight = 100; // 假设每个格子的高度为100px
    const deltaSize = Math.round(deltaX / cellWidth);
    const deltaHeight = Math.round(deltaY / cellHeight);
    
    const course = resizingCourse.value;
    
    if (direction === 'right') {
      // 右侧边缘调整 - 只调整宽度
      const newSize = Math.max(1, Math.min(5, resizeStartSize.value + deltaSize));
      
      // 检查新尺寸是否可用
      if (canResizeCourse(course, newSize, course.height)) {
        course.size = newSize;
      }
    } else if (direction === 'left') {
      // 左侧边缘调整 - 调整宽度和起始列
      const deltaCol = deltaSize;
      const newStartCol = Math.max(0, Math.min(4, resizeStartCol.value + deltaCol));
      const newSize = Math.max(1, Math.min(5, resizeStartSize.value - deltaSize));
      
      // 检查新位置和尺寸是否可用
      if (newStartCol + newSize <= 5) {
        if (canResizeCourse(course, newSize, course.height, newStartCol, course.row)) {
          course.startCol = newStartCol;
          course.size = newSize;
        }
      }
    } else if (direction === 'bottom') {
      // 底部边缘调整 - 只调整高度
      const newHeight = Math.max(1, Math.min(2, resizeStartHeight.value + deltaHeight));
      
      // 检查新尺寸是否可用
      if (canResizeCourse(course, course.size, newHeight)) {
        course.height = newHeight;
      }
    } else if (direction === 'top') {
      // 顶部边缘调整 - 调整高度和起始行
      const deltaRow = deltaHeight;
      const newStartRow = Math.max(0, Math.min(1, resizeStartRow.value + deltaRow));
      const newHeight = Math.max(1, Math.min(2, resizeStartHeight.value - deltaHeight));
      
      // 检查新位置和尺寸是否可用
      if (newStartRow + newHeight <= 2) {
        if (canResizeCourse(course, course.size, newHeight, course.startCol, newStartRow)) {
          course.row = newStartRow;
          course.height = newHeight;
        }
      }
    } else if (direction === 'top-left') {
      // 左上角调整 - 调整宽度、高度、起始列和起始行
      const deltaCol = deltaSize;
      const deltaRow = deltaHeight;
      const newStartCol = Math.max(0, Math.min(4, resizeStartCol.value + deltaCol));
      const newStartRow = Math.max(0, Math.min(1, resizeStartRow.value + deltaRow));
      const newSize = Math.max(1, Math.min(5, resizeStartSize.value - deltaSize));
      const newHeight = Math.max(1, Math.min(2, resizeStartHeight.value - deltaHeight));
      
      if (newStartCol + newSize <= 5 && newStartRow + newHeight <= 2) {
        if (canResizeCourse(course, newSize, newHeight, newStartCol, newStartRow)) {
          course.startCol = newStartCol;
          course.row = newStartRow;
          course.size = newSize;
          course.height = newHeight;
        }
      }
    } else if (direction === 'top-right') {
      // 右上角调整 - 调整宽度、高度和起始行
      const deltaRow = deltaHeight;
      const newStartRow = Math.max(0, Math.min(1, resizeStartRow.value + deltaRow));
      const newSize = Math.max(1, Math.min(5, resizeStartSize.value + deltaSize));
      const newHeight = Math.max(1, Math.min(2, resizeStartHeight.value - deltaHeight));
      
      if (course.startCol + newSize <= 5 && newStartRow + newHeight <= 2) {
        if (canResizeCourse(course, newSize, newHeight, course.startCol, newStartRow)) {
          course.row = newStartRow;
          course.size = newSize;
          course.height = newHeight;
        }
      }
    } else if (direction === 'bottom-left') {
      // 左下角调整 - 调整宽度、高度和起始列
      const deltaCol = deltaSize;
      const newStartCol = Math.max(0, Math.min(4, resizeStartCol.value + deltaCol));
      const newSize = Math.max(1, Math.min(5, resizeStartSize.value - deltaSize));
      const newHeight = Math.max(1, Math.min(2, resizeStartHeight.value + deltaHeight));
      
      if (newStartCol + newSize <= 5 && course.row + newHeight <= 2) {
        if (canResizeCourse(course, newSize, newHeight, newStartCol, course.row)) {
          course.startCol = newStartCol;
          course.size = newSize;
          course.height = newHeight;
        }
      }
    } else if (direction === 'bottom-right') {
      // 右下角调整 - 调整宽度和高度
      const newSize = Math.max(1, Math.min(5, resizeStartSize.value + deltaSize));
      const newHeight = Math.max(1, Math.min(2, resizeStartHeight.value + deltaHeight));
      
      if (course.startCol + newSize <= 5 && course.row + newHeight <= 2) {
        if (canResizeCourse(course, newSize, newHeight)) {
          course.size = newSize;
          course.height = newHeight;
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
      for (let r = course.row; r < course.row + course.height; r++) {
        for (let col = course.startCol; col < course.startCol + course.size; col++) {
          status[r][col] = true;
        }
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
 * @param {number} height - 课程高度
 * @returns {boolean} 是否可用
 */
const isPositionAvailable = (row: number, startCol: number, size: number, height: number = 1): boolean => {
  if (startCol + size > 5 || row + height > 2) return false;
  
  for (let r = row; r < row + height; r++) {
    for (let col = startCol; col < startCol + size; col++) {
      if (gridStatus.value[r][col]) {
        return false;
      }
    }
  }
  return true;
};

/**
 * 检查课程是否可以调整到新尺寸
 * @param {Course} course - 要调整的课程
 * @param {number} newSize - 新宽度
 * @param {number} newHeight - 新高度
 * @param {number} newStartCol - 新起始列
 * @param {number} newStartRow - 新起始行
 * @returns {boolean} 是否可以调整
 */
const canResizeCourse = (course: Course, newSize: number, newHeight: number = course.height, newStartCol: number = course.startCol, newStartRow: number = course.row): boolean => {
  if (newSize < 1 || newSize > 5) return false;
  if (newHeight < 1 || newHeight > 2) return false;
  if (newStartCol + newSize > 5) return false;
  if (newStartRow + newHeight > 2) return false;
  
  for (let r = newStartRow; r < newStartRow + newHeight; r++) {
    for (let col = newStartCol; col < newStartCol + newSize; col++) {
      // 检查是否有其他课程占用这个位置
      let isOccupied = false;
      for (const other of courses.value) {
        if (other && other !== course) {
          if (r >= other.row && r < other.row + other.height && col >= other.startCol && col < other.startCol + other.size) {
            isOccupied = true;
            break;
          }
        }
      }
      if (isOccupied) {
        return false;
      }
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
  
  // 计算最大可用宽度和高度
  let maxSize = 5 - col;
  let maxHeight = 2 - row;
  
  // 找到最大可用的宽度
  for (let size = 1; size <= maxSize; size++) {
    if (!isPositionAvailable(row, col, size, 1)) {
      maxSize = size - 1;
      break;
    }
  }
  
  // 找到最大可用的高度
  for (let height = 1; height <= maxHeight; height++) {
    if (!isPositionAvailable(row, col, 1, height)) {
      maxHeight = height - 1;
      break;
    }
  }
  
  if (maxSize === 0 || maxHeight === 0) {
    alert('该位置无法添加课程');
    return;
  }
  
  // 提示用户选择课程宽度
  const sizeInput = prompt(`请输入课程宽度 (1-${maxSize})`, '1');
  const size = parseInt(sizeInput || '1');
  
  if (isNaN(size) || size < 1 || size > maxSize) {
    alert(`无效的课程宽度，请输入 1-${maxSize} 之间的数字`);
    return;
  }
  
  // 提示用户选择课程高度
  const heightInput = prompt(`请输入课程高度 (1-${maxHeight})`, '1');
  const height = parseInt(heightInput || '1');
  
  if (isNaN(height) || height < 1 || height > maxHeight) {
    alert(`无效的课程高度，请输入 1-${maxHeight} 之间的数字`);
    return;
  }
  
  // 检查选择的尺寸是否可用
  if (!isPositionAvailable(row, col, size, height)) {
    alert('选择的尺寸无法放置在该位置');
    return;
  }
  
  const name = prompt('请输入课程名称', '新课程');
  if (!name) return;
  
  courses.value.push({
    id: nextId.value++,
    name,
    size,
    height,
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
    if (course && course !== draggedCourse.value) {
      if (row >= course.row && row < course.row + course.height && col >= course.startCol && col < course.startCol + course.size) {
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
  // 检查行是否重叠
  const rowOverlap = !(course1.row + course1.height <= course2.row || course2.row + course2.height <= course1.row);
  // 检查列是否重叠
  const colOverlap = !(course1.startCol + course1.size <= course2.startCol || course2.startCol + course2.size <= course1.startCol);
  return rowOverlap && colOverlap;
};

/**
 * 检查两个课程是否紧凑排列
 * @param {Course} course1 - 第一个课程
 * @param {Course} course2 - 第二个课程
 * @returns {boolean} 是否紧凑排列
 */
const isCompact = (course1: Course, course2: Course): boolean => {
  if (course1.row !== course2.row) return false;
  const end1 = course1.startCol + course1.size;
  const start2 = course2.startCol;
  const end2 = course2.startCol + course2.size;
  const start1 = course1.startCol;
  
  // 检查两个课程是否相邻且紧凑排列
  return (end1 === start2 && course1.size + course2.size <= 5) || (end2 === start1 && course1.size + course2.size <= 5);
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
  if (row < 0 || row + course.height > 2) return false;

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
  return isPositionAvailable(row, col, draggedCourse.value.size, draggedCourse.value.height);
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

    // 检查是否满足紧凑排列的条件
    const isCompactArrangement = isCompact(dragged, targetCourse) && (dragged.size > 1 || targetCourse.size > 1);

    if (isCompactArrangement) {
      // 紧凑排列的特殊处理：直接调换顺序并保持紧凑
      const minStartCol = Math.min(dragged.startCol, targetCourse.startCol);
      const totalSize = dragged.size + targetCourse.size;
      
      // 确定调换后的顺序
      if (dragged.startCol < targetCourse.startCol) {
        // 原顺序：dragged 在 targetCourse 左侧
        // 调换后：targetCourse 在 dragged 左侧
        targetCourse.startCol = minStartCol;
        dragged.startCol = minStartCol + targetCourse.size;
      } else {
        // 原顺序：targetCourse 在 dragged 左侧
        // 调换后：dragged 在 targetCourse 左侧
        dragged.startCol = minStartCol;
        targetCourse.startCol = minStartCol + dragged.size;
      }
    } else {
      // 常规处理
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
    }
  } else {
    // 检查新位置是否可用（需要连续的size个格子和height行）
    if (!isPositionAvailable(row, col, dragged.size, dragged.height)) {
      alert('该位置无法放置课程，需要连续的' + dragged.size + '个格子和' + dragged.height + '行');
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
              width: `calc((${course.size} * (100% - 40px)) / 5 + ${(course.size - 1) * 10}px)`,
              height: course.height === 1 ? '100px' : 'calc(200px + 15px)'
            }"
            :draggable="!resizingCourse"
            @dragstart="dragStart(course)"
            @dragover="dragOver($event, course.row, course.startCol)"
            @drop="drop(course.row, course.startCol)"
          >
            <div class="resize-handle top" @mousedown="startResize(course, $event, 'top')"></div>
            <div class="resize-handle left" @mousedown="startResize(course, $event, 'left')"></div>
            <div class="course-content">
              <span>{{ course.name }}</span>
              <button class="remove-btn" @click.stop="removeCourse(course)">×</button>
            </div>
            <div class="resize-handle right" @mousedown="startResize(course, $event, 'right')"></div>
            <div class="resize-handle bottom" @mousedown="startResize(course, $event, 'bottom')"></div>
            <div class="resize-handle top-left" @mousedown="startResize(course, $event, 'top-left')"></div>
            <div class="resize-handle top-right" @mousedown="startResize(course, $event, 'top-right')"></div>
            <div class="resize-handle bottom-left" @mousedown="startResize(course, $event, 'bottom-left')"></div>
            <div class="resize-handle bottom-right" @mousedown="startResize(course, $event, 'bottom-right')"></div>
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
  background-color: rgba(255, 255, 255, 0.3);
  transition: background-color 0.2s ease;
}

.resize-handle.left,
.resize-handle.right {
  top: 0;
  bottom: 0;
  width: 10px;
  cursor: ew-resize;
}

.resize-handle.top,
.resize-handle.bottom {
  left: 0;
  right: 0;
  height: 10px;
  cursor: ns-resize;
}

.resize-handle.top-left,
.resize-handle.top-right,
.resize-handle.bottom-left,
.resize-handle.bottom-right {
  width: 10px;
  height: 10px;
}

.resize-handle.top-left,
.resize-handle.bottom-right {
  cursor: nwse-resize;
}

.resize-handle.top-right,
.resize-handle.bottom-left {
  cursor: ne-resize;
}

.resize-handle.left {
  left: 0;
}

.resize-handle.right {
  right: 0;
}

.resize-handle.top {
  top: 0;
}

.resize-handle.bottom {
  bottom: 0;
}

.resize-handle.top-left {
  top: 0;
  left: 0;
}

.resize-handle.top-right {
  top: 0;
  right: 0;
}

.resize-handle.bottom-left {
  bottom: 0;
  left: 0;
}

.resize-handle.bottom-right {
  bottom: 0;
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