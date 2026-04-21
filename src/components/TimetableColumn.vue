<script setup lang="ts">
import { ref, computed } from 'vue';

/**
 * 课程类型定义
 * @property {number} id - 课程唯一标识符
 * @property {string} name - 课程名称
 * @property {number} size - 课程高度，1-3个格子
 * @property {number} startRow - 课程起始行，0-2
 */
interface Course {
  id: number;
  name: string;
  size: number; // 1-3
  startRow: number; // 0-2
}

// 课程表数据
const courses = ref<Course[]>([]);
const nextId = ref(1);

// 拖拽相关
const draggedCourse = ref<Course | null>(null);
const dragTarget = ref<number | null>(null);

/**
 * 开始拖拽课程
 * @param {Course} course - 要拖拽的课程
 */
const dragStart = (course: Course) => {
  if (resizingCourse.value) return;
  draggedCourse.value = course;
};

/**
 * 拖拽经过格子时的处理
 * @param {DragEvent} event - 拖拽事件
 * @param {number} row - 格子所在行
 */
const dragOver = (event: DragEvent, row: number) => {
  event.preventDefault();
  dragTarget.value = row;
};

/**
 * 拖拽离开格子时的处理
 */
const dragLeave = () => {
  dragTarget.value = null;
};

// 调整尺寸相关
const resizingCourse = ref<Course | null>(null);
const resizingElement = ref<HTMLElement | null>(null);
const resizeStartY = ref(0);
const resizeStartSize = ref(0);
const resizeStartRow = ref(0);

/**
 * 开始调整课程尺寸
 * @param {Course} course - 要调整尺寸的课程
 * @param {MouseEvent} event - 鼠标事件
 * @param {'top' | 'bottom'} direction - 调整方向，默认为底部
 */
const startResize = (course: Course, event: MouseEvent, direction: 'top' | 'bottom' = 'bottom') => {
  event.stopPropagation();
  event.preventDefault();
  
  resizingCourse.value = course;
  resizeStartY.value = event.clientY;
  resizeStartSize.value = course.size;
  resizeStartRow.value = course.startRow;
  
  // 禁用课程元素的 draggable 属性，避免触发拖动位置事件
  const target = event.currentTarget as HTMLElement;
  const courseElement = target.closest('.course') as HTMLElement;
  if (courseElement) {
    resizingElement.value = courseElement;
    courseElement.draggable = false;
  }
  
  const handleMouseMove = (e: MouseEvent) => {
    e.stopPropagation();
    e.preventDefault();
    
    if (!resizingCourse.value) return;
    
    const deltaY = e.clientY - resizeStartY.value;
    const cellHeight = 100; // 每个格子的高度为100px
    const deltaSize = Math.round(deltaY / cellHeight);
    
    if (direction === 'bottom') {
      const newSize = Math.max(1, Math.min(3, resizeStartSize.value + deltaSize));
      
      if (canResizeCourse(resizingCourse.value, newSize)) {
        resizingCourse.value.size = newSize;
      }
    } else {
      // 顶部边缘调整，方向与底部相反
      const newSize = Math.max(1, Math.min(3, resizeStartSize.value - deltaSize));
      const newStartRow = Math.max(0, Math.min(2, resizeStartRow.value + deltaSize));
      
      if (newStartRow + newSize <= 3) {
        let canResize = true;
        for (let row = newStartRow; row < newStartRow + newSize; row++) {
          let isOccupied = false;
          for (const other of courses.value) {
            if (other && other !== resizingCourse.value) {
              if (row >= other.startRow && row < other.startRow + other.size) {
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
          resizingCourse.value.startRow = newStartRow;
          resizingCourse.value.size = newSize;
        }
      }
    }
  };
  
  const handleMouseUp = (e: MouseEvent) => {
    e.stopPropagation();
    e.preventDefault();
    
    // 恢复课程元素的 draggable 属性
    if (resizingElement.value) {
      resizingElement.value.draggable = true;
      resizingElement.value = null;
    }
    resizingCourse.value = null;
    document.removeEventListener('mousemove', handleMouseMove);
    document.removeEventListener('mouseup', handleMouseUp);
  };
  
  document.addEventListener('mousemove', handleMouseMove);
  document.addEventListener('mouseup', handleMouseUp);
};

/**
 * 计算每个格子的状态
 * @returns {boolean[]} 3格的占用状态数组
 */
const gridStatus = computed(() => {
  const status = Array(3).fill(false);
  
  courses.value.forEach(course => {
    if (course && course !== draggedCourse.value && course !== resizingCourse.value) {
      for (let row = course.startRow; row < course.startRow + course.size; row++) {
        status[row] = true;
      }
    }
  });
  
  return status;
});

const isPositionAvailable = (startRow: number, size: number): boolean => {
  if (startRow + size > 3) return false;
  
  for (let row = startRow; row < startRow + size; row++) {
    if (gridStatus.value[row]) {
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
  if (newSize < 1 || newSize > 3) return false;
  if (course.startRow + newSize > 3) return false;
  
  for (let row = course.startRow; row < course.startRow + newSize; row++) {
    let isOccupied = false;
    for (const other of courses.value) {
      if (other && other !== course) {
        if (row >= other.startRow && row < other.startRow + other.size) {
          isOccupied = true;
          break;
        }
      }
    }
    if (isOccupied) return false;
  }
  return true;
};

/**
 * 添加课程
 * @param {number} row - 行号
 */
const addCourse = (row: number) => {
  if (gridStatus.value[row]) {
    alert('该位置已被占用');
    return;
  }
  
  let maxSize = 3 - row;
  for (let size = 1; size <= maxSize; size++) {
    if (!isPositionAvailable(row, size)) {
      maxSize = size - 1;
      break;
    }
  }
  
  if (maxSize === 0) {
    alert('该位置无法添加课程');
    return;
  }
  
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
    startRow: row
  });
};

/**
 * 获取目标位置上的课程
 * @param {number} row - 行号
 * @returns {Course | null} 目标位置上的课程
 */
const getCourseAtPosition = (row: number): Course | null => {
  for (const course of courses.value) {
    if (course && course !== draggedCourse.value) {
      if (row >= course.startRow && row < course.startRow + course.size) {
        return course;
      }
    }
  }
  return null;
};

/**
 * 检查两个课程的位置是否重叠
 */
const hasOverlap = (course1: Course, course2: Course): boolean => {
  const start1 = course1.startRow;
  const end1 = course1.startRow + course1.size;
  const start2 = course2.startRow;
  const end2 = course2.startRow + course2.size;
  return start1 < end2 && start2 < end1;
};

/**
 * 检查课程是否可以放置在某个位置
 */
const canPlaceCourse = (course: Course, startRow: number, excludeCourse: Course | null): boolean => {
  if (startRow < 0 || startRow + course.size > 3) return false;

  const tempCourse: Course = { ...course, startRow };
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
 */
const isDroppable = (row: number): boolean => {
  if (!draggedCourse.value) return false;
  return isPositionAvailable(row, draggedCourse.value.size);
};

/**
 * 放置课程
 */
const drop = (row: number) => {
  if (!draggedCourse.value) return;

  const dragged = draggedCourse.value;
  const targetCourse = getCourseAtPosition(row);

  if (targetCourse) {
    const draggedNewRow = targetCourse.startRow;
    const targetNewRow = dragged.startRow;

    const canDraggedMove = canPlaceCourse(dragged, draggedNewRow, targetCourse);
    const canTargetMove = canPlaceCourse(targetCourse, targetNewRow, dragged);

    if (canDraggedMove && canTargetMove) {
      const tempRow = dragged.startRow;
      dragged.startRow = draggedNewRow;
      targetCourse.startRow = tempRow;

      // 检查调换后是否存在重叠
      if (hasOverlap(dragged, targetCourse)) {
        // 找出尺寸较小的课程
        const smallerCourse = dragged.size <= targetCourse.size ? dragged : targetCourse;
        const largerCourse = dragged.size > targetCourse.size ? dragged : targetCourse;

        // 计算重叠部分，将较小的课程往下推
        let newStartRow = largerCourse.startRow + largerCourse.size;
        // 确保新位置不会超出课程表范围
        if (newStartRow + smallerCourse.size <= 3) {
          smallerCourse.startRow = newStartRow;
        }
      }
    } else {
      alert('这两个课程无法互换位置');
    }
  } else {
    if (!isPositionAvailable(row, dragged.size)) {
      alert('该位置无法放置课程，需要连续的' + dragged.size + '个格子');
      return;
    }

    dragged.startRow = row;
  }

  draggedCourse.value = null;
};

/**
 * 移除课程
 */
const removeCourse = (course: Course) => {
  courses.value = courses.value.filter(c => c.id !== course.id);
};
</script>

<template>
  <div class="timetable-column-container">
    <h2>课程表（竖向）</h2>
    <div class="timetable-column">
      <div 
        v-for="(_, rowIndex) in 3" 
        :key="rowIndex" 
        class="timetable-cell"
        :class="{ 
          occupied: gridStatus[rowIndex],
          droppable: isDroppable(rowIndex)
        }"
        @click="!gridStatus[rowIndex] && addCourse(rowIndex)"
        @dragover="dragOver($event, rowIndex)"
        @dragleave="dragLeave"
        @drop="drop(rowIndex)"
      ></div>
      
      <!-- 渲染课程 -->
      <template v-for="course in courses" :key="course.id">
        <div 
          class="course"
          :style="{
            top: `calc(${course.startRow} * (100% - 20px) / 3 + ${course.startRow * 10}px)`,
            height: `calc((${course.size} * (100% - 20px)) / 3 + ${(course.size - 1) * 10}px)`
          }"
          draggable="true"
          @dragstart="dragStart(course)"
          @dragover="dragOver($event, course.startRow)"
          @drop="drop(course.startRow)"
        >
          <div class="resize-handle top" @mousedown="startResize(course, $event, 'top')"></div>
          <div class="course-content">
            <span>{{ course.name }}</span>
            <button class="remove-btn" @click.stop="removeCourse(course)">×</button>
          </div>
          <div class="resize-handle bottom" @mousedown="startResize(course, $event, 'bottom')"></div>
        </div>
      </template>
    </div>
    <div class="instructions">
      <p>点击空白格子添加课程</p>
      <p>拖拽课程调整位置</p>
      <p>点击课程上的 × 移除课程</p>
    </div>
  </div>
</template>

<style scoped>
.timetable-column-container {
  max-width: 300px;
  margin: 0 auto;
  padding: 20px;
}

h2 {
  text-align: center;
  color: #42b983;
}

.timetable-column {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin: 20px 0;
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
  left: 0;
  right: 0;
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
  left: 0;
  right: 0;
  height: 10px;
  background-color: rgba(255, 255, 255, 0.3);
  cursor: ns-resize;
  transition: background-color 0.2s ease;
}

.resize-handle.top {
  top: 0;
}

.resize-handle.bottom {
  bottom: 0;
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
