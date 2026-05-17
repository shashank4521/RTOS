# RTOS
What is a Mutex?
A Mutex is essentially a specialized locking mechanism used to synchronize access to a resource. Think of it as a unique physical key to a single-occupancy restroom.

If a task wants to use the shared resource (e.g., write to a serial terminal, modify a global variable, or use an I2C/SPI bus), it must first "take" (lock) the Mutex key.

If another task tries to access the resource while the key is taken, the RTOS puts that second task into a blocked state (sleeping), forcing it to wait.

Once the first task finishes, it "gives" (unlocks) the Mutex back, allowing the next waiting task to wake up and safely utilize the resource.

Without a Mutex, tasks can interrupt each other mid-execution, leading to a classic bug known as a Race Condition or data corruption.

<img width="370" height="122" alt="image" src="https://github.com/user-attachments/assets/3e58215a-d714-4b50-bc11-362f2545e0a9" />





<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/280363cd-9747-40a5-90b1-74c486a56894" />





// A shared global variable or a hardware peripheral like Serial
int sharedCounter = 0;

void vTask1(void *pvParameters) {
  while(1) {
    // Task 1 attempts to print a cohesive message
    Serial.print("Task 1 is running: ");
    sharedCounter++;
    Serial.println(sharedCounter);
    
    vTaskDelay(pdMS_TO_TICKS(5)); // Yield control briefly
  }
}

void vTask2(void *pvParameters) {
  while(1) {
    // Task 2 also attempts to print its message
    Serial.print("--- Task 2 Here! Counter is: ");
    Serial.println(sharedCounter);
    
    vTaskDelay(pdMS_TO_TICKS(5));
  }
}

void setup() {
  Serial.begin(115200);
  
  // Creating tasks with equal priority
  xTaskCreate(vTask1, "Task 1", 2048, NULL, 1, NULL);
  xTaskCreate(vTask2, "Task 2", 2048, NULL, 1, NULL);
}

void loop() {
  // Empty when using an RTOS architecture
}


Task 1 is running: --- Task 2 Here! Counter is: 1
2
Task 1 is running: 3
--- Task 2 Here! Task 1 is running: Counter is: 3
4


2. Code WITH a Mutex (Protected Resource)
To fix this, we implement a SemaphoreHandle_t configured as a Mutex. This forces both tasks to respect boundaries and wait their turn.


#include <Arduino.h> // If using ESP32 Arduino framework

// Declare a global handle for the Mutex
SemaphoreHandle_t xMutex;
int sharedCounter = 0;

void vTask1(void *pvParameters) {
  while(1) {
    // Try to take the Mutex. Wait indefinitely (portMAX_DELAY) if it's taken.
    if (xSemaphoreTake(xMutex, portMAX_DELAY) == pdTRUE) {
      
      // CRITICAL SECTION: Only Task 1 can be inside this block of code right now
      Serial.print("Task 1 is running: ");
      sharedCounter++;
      Serial.println(sharedCounter);
      
      // Always give the Mutex back when finished!
      xSemaphoreGive(xMutex);
    }
    
    vTaskDelay(pdMS_TO_TICKS(5)); 
  }
}

void vTask2(void *pvParameters) {
  while(1) {
    // Try to take the Mutex to protect its own printing sequence
    if (xSemaphoreTake(xMutex, portMAX_DELAY) == pdTRUE) {
      
      // CRITICAL SECTION
      Serial.print("--- Task 2 Here! Counter is: ");
      Serial.println(sharedCounter);
      
      xSemaphoreGive(xMutex);
    }
    
    vTaskDelay(pdMS_TO_TICKS(5));
  }
}

void setup() {
  Serial.begin(115200);
  
  // Create the Mutex before starting the tasks
  xMutex = xSemaphoreCreateMutex();
  
  if (xMutex != NULL) {
    // Create tasks only if the Mutex was successfully initialized
    xTaskCreate(vTask1, "Task 1", 2048, NULL, 1, NULL);
    xTaskCreate(vTask2, "Task 2", 2048, NULL, 1, NULL);
  }
}

void loop() {
}


Task 1 is running: 1
--- Task 2 Here! Counter is: 1
Task 1 is running: 2
--- Task 2 Here! Counter is: 2
Task 1 is running: 3
--- Task 2 Here! Counter is: 3

conclusion :
1. The RTOS is still switching between tasks
Because both tasks have the same priority, the RTOS scheduler is constantly splitting CPU time between them (a mechanism called time-slicing).

The scheduler might let Task 1 run for 1 millisecond, then pause it, let Task 2 run for 1 millisecond, pause it, and go back to Task 1.

2. What happens inside the code execution loop
Let's look at the timeline of how they interact with the Mutex:

Step 1: Task 1 runs and reaches xSemaphoreTake(xMutex, portMAX_DELAY). The Mutex is free, so Task 1 locks it and starts printing "Task 1 is running: ".

Step 2 (The Interruption): Mid-sentence, the RTOS timer clicks. The scheduler pauses Task 1 right there and switches over to Task 2.

Step 3 (The Block): Task 2 wakes up and tries to execute xSemaphoreTake(xMutex, portMAX_DELAY). It sees that the Mutex is locked by Task 1.

Step 4 (The Waiting Room): Instead of continuing and messing up the serial print, Task 2 tells the scheduler, "I can't proceed without this key. Put me to sleep until it's free." The RTOS immediately takes Task 2 out of the running queue and puts it into a Blocked state.

Step 5 (Finishing the Job): Since Task 2 is asleep, the scheduler hands control right back to Task 1. Task 1 finishes printing the counter value and then calls xSemaphoreGive(xMutex).

Step 6 (The Handover): The moment Task 1 unlocks the Mutex, the RTOS wakes up Task 2. When the next time-slice occurs, Task 2 will successfully lock the Mutex, execute its own print cleanly, and give it back.

Summary
They are both running at the same time in the grand scheme of things. However, the Mutex acts like a safety gate. It ensures that the specific block of code between Take and Give (called the Critical Section) can only be executed by one task at a time, preventing them from cutting each other off mid-print.


                     
