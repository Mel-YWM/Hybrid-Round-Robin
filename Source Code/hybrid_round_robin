#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include <string.h>
#include <ctype.h>

/*
Process / File scheduler implements round robin (RR) with dynamic time quantums & shortest job first approach (SJF).
>>RR default time quantum == 5. 

If only 1 item in queue and process in queue has a burst time lower than default, 
>> time quantum == process's remaining burst time.
>> process burst time > default time quantum (5), default time quantum used instead.

If a new process enters queue and more than 1 process in queue.
    (1) Continue to process current process until time quantum elapses.
    (2) Iterate through queue to find process with (a) lowest burst time. Use its burst time as time quantum. (SJF)
        >> Note if new burst time > 5, time quantum == 5.
        >> i.e., RR + SJF.
    (3) If all processes has the same remaining burst time, use FIFO approach
        >> i.e., file with lowest arrival time will be processed instead. 
*/

int ITEMS_IN_QUEUE = 0;
int DEFAULT_TIME_QUANTUM = 5; // arbritary default value
int TIME_ELAPSED = 0;
char line[100];

typedef struct fileAttribute
{
    int arrivalTime; // time process enters ready state
    int burstTime;   // time required for process
    int waitingTime; // total time process is in ready state
    int turnAroundTime; // time till completion since arrival
    int backUpBurstTime; // when 0, process is deemed to have completed
    int timeAddedToQueue; // time process gets added into queue (SHould be same as arrival time)
    int fileID;           // corresponds to index it is stored in
    int responseTime;     // time process gets passed to CPU
    int timeStopped;

    bool hasArrived;   // if time-elapsed, file/proceess deemed to have arrived
    bool hasCompleted; // ignore once completed
    bool inQueue;      // if its in the ready queue
    bool processing;   // if it is currently being processed
    bool saved;        // bool to check if time that process changes is saved
    bool calculated;   // bool to check if waiting time has been calculated
    bool startedButNotComp; // process requeued, started but not completed
} fileAttribute;
// Struct that stores basic file attributes, increases modularity and allows other aspects of file to be stored
// e.g., priority, type, group etc.

typedef struct readyQueueNode
{
    // ready queue implemented as a linked list 
    fileAttribute file;
    struct readyQueueNode *next;
} readyQueueNode;

int getNumberOfFiles(FILE *fp);
void printFileTable(fileAttribute fileTable[], int numberOfFiles);
void populateQueueIfArrive(int numberOfFiles, fileAttribute fileTable[], readyQueueNode *readyQueue);
void AddToReadyQueue(fileAttribute fileTableEntry, readyQueueNode *readyQueue);
void printReadyQueueEntries(readyQueueNode *readyQueue);
void HybridRoundRobin(int TIME_QUANTUM, readyQueueNode *readyQueue, fileAttribute fileTable[], fileAttribute completedFileTable[], int numberOfFiles, int *processCompleted, int *completedIndex);
void UpdateWaitingTimes(readyQueueNode *readyQueue);
int getLowestRemainingBurstTime(readyQueueNode *readyQueue);
int CountUncompletedProcessesInQueue(readyQueueNode *readyQueue);
int validate(char* value);
float GetAverageTurnaroundTime(fileAttribute completedFileTable[], int numberOfFiles);
float GetAverageWaitingTime(fileAttribute completedFileTable[], int numberOfFiles);
float GetMinWaitingTime(fileAttribute completedFileTable[], int numberOfFiles);
float GetMaxWaitingTime(fileAttribute completedFileTable[], int numberOfFiles);
float GetMinTurnAroundTime(fileAttribute completedFileTable[], int numberOfFiles);
float GetMaxTurnAroundTime(fileAttribute completedFileTable[], int numberOfFiles);
void printOrderOfCompletion(fileAttribute completedFileTable[], int numberOfFiles);
void freeAllocatedMemory(readyQueueNode *readyQueue);

int main(int argc, char* argv[])
{
    if (argc != 2) {
        printf("Invalid usage: <./assignment> <filename.txt>\n");
        exit(1);
    }

    FILE *fp = fopen(argv[1], "r");
    if (fp == NULL)
    {
        printf("<%s> does not exist!\n", argv[1]);
        exit(1);
    }

    int TIME_QUANTUM = DEFAULT_TIME_QUANTUM; // Intial time_quantum == Default time quantum (5)
    int numberOfFiles = 0;

    // Obtains number of files and creates an array to store their file attributes
    numberOfFiles = getNumberOfFiles(fp);
    fileAttribute fileTable[numberOfFiles];
    fileAttribute completedFileTable[numberOfFiles];
    int completedIndex = 0; // maintains order in which processes were completed

    // Initially create a ready linked list that can contain 1 node, each node contains a fileTable entry and next
    readyQueueNode *readyQueue = malloc(sizeof(readyQueueNode));
   
    int processesInQueue = 0;

    if (readyQueue == NULL)
    {
        printf("Failed to allocate memory\n");
        exit(1);
    }

    // Initializing file table contents 
    // fileTable[i] stores parsed data from text file e.g., burst  + waiting time
    // completedFileTable[i] stores computed burst, waiting, turnaround times. 
    for (int i = 0; i < numberOfFiles; i++)
    {
        fileTable[i].arrivalTime = -1;
        fileTable[i].burstTime = -1;
        fileTable[i].waitingTime = 0;
        fileTable[i].turnAroundTime = 0;
        fileTable[i].backUpBurstTime = -1;
        fileTable[i].fileID = i; // fileID also == index it is stored within fileTable[]
        fileTable[i].timeAddedToQueue = 0;
        fileTable[i].responseTime = -1;
        fileTable[i].timeStopped = 0;

        fileTable[i].hasArrived = false;
        fileTable[i].hasCompleted = false;
        fileTable[i].inQueue = false;
        fileTable[i].processing = false;
        fileTable[i].saved = false;
        fileTable[i].calculated = true;
        fileTable[i].startedButNotComp = false;

        completedFileTable[i].arrivalTime = -1;
        completedFileTable[i].burstTime = -1;
        completedFileTable[i].waitingTime = 0;
        completedFileTable[i].turnAroundTime = 0;
        completedFileTable[i].backUpBurstTime = -1;
        completedFileTable[i].fileID = i; 
        completedFileTable[i].timeAddedToQueue = 0;
        completedFileTable[i].responseTime = -1;

        completedFileTable[i].hasArrived = false;
        completedFileTable[i].hasCompleted = false;
        completedFileTable[i].inQueue = false;
        completedFileTable[i].processing = false;
    }

    // Obtain arrival and burst time from text file
    int i = 0, j = 0, k = 0, counter = 0;
    int bufferLength = 255;
    char buffer[255];//buffer to hold file input
    while(fgets(buffer, bufferLength, fp)) { //retrieve file input line by line
        int v = 0;
        char line[6];
        char *ptr = strtok(buffer, " \t");
        while (v<2)
        {
            strcpy(line,ptr); //copy string from input to line
            char *newptr = line;
            newptr[strcspn(newptr, "\n")] = 0; //remove newline \n from input
            int check = validate(newptr); //ensure only positive integer values
            if(check == 0){
                printf("Invalid input detected at line %d \n",counter+1);
                exit(0);
            }
            if(v==0){
                int j = (int)strtol(newptr, (char **)NULL, 10);
                fileTable[i].arrivalTime = j; //store arrival time
            }else if(v==1){
                int k = (int)strtol(newptr, (char **)NULL, 10);
                fileTable[i].burstTime = k;
                fileTable[i].backUpBurstTime = k; //store burst time
            }
            ptr = strtok(NULL, " \t"); //move to next value in line
            v++;
        }
        i++;
        counter++;
        if(counter == numberOfFiles){ //to ensure we do not accept inputs after newline encountered
                break;
            }
        }

        
    int processesCompleted = 0;
    while (processesCompleted != numberOfFiles)
    {
        // Each while-loop iteration increments TIME_ELAPSED by 1, or by TIME_QUANTUM if round robin executes.
        // At each while-loop iteration, check the arrival time, if arrival time is greater or equals to time_elapsed
        // add this file into the ready queue
        populateQueueIfArrive(numberOfFiles, fileTable, readyQueue);
        HybridRoundRobin(TIME_QUANTUM, readyQueue, fileTable, completedFileTable, numberOfFiles, &processesCompleted, &completedIndex);
        UpdateWaitingTimes(readyQueue);
    }
    float averageTurnaroundTime = GetAverageTurnaroundTime(completedFileTable, numberOfFiles);
    float maxTurn = GetMaxTurnAroundTime(completedFileTable, numberOfFiles);
    float averageWaitingTime = GetAverageWaitingTime(completedFileTable, numberOfFiles);
    float maxWait = GetMaxWaitingTime(completedFileTable, numberOfFiles);
    float minWait = GetMinWaitingTime(completedFileTable, numberOfFiles);
    float minTurn = GetMinTurnAroundTime(completedFileTable, numberOfFiles);
    printOrderOfCompletion(completedFileTable, numberOfFiles);
    fclose(fp);
    freeAllocatedMemory(readyQueue);
    exit(1);
}

int getNumberOfFiles(FILE *fp)
{
    // Function returns number of files present in sample.txt.
    // Sample.txt - last line must be an empty line (as per specifications)
    int count = 0;
    while(fgets(line, sizeof(line), fp) != NULL) 
    {
        if (line[0] != '\n')
        { 
            count++;
        }
        else
        {
            break;
        }
    }
    fseek(fp, 0, SEEK_SET);
    return count;
}

void printFileTable(fileAttribute fileTable[], int numberOfFiles)
{
    // Accepts fileTable / completedfileTable and number of files as arguments
    // Helper function to check contents of file table
    printf("\t\tFID\tArrival Time\tBurst Time\tWaiting Time\tTurnaround Time\t BackUp BurstTime\tResponse Time\n");
    for (int i = 0; i < numberOfFiles; i++)
    {
        printf("Filetable[%i]:\t%i\t%i\t\t%i\t\t%i\t\t%i\t\t %i\t\t\t%i\n", i, fileTable[i].fileID, fileTable[i].arrivalTime, fileTable[i].burstTime, fileTable[i].waitingTime, fileTable[i].turnAroundTime, fileTable[i].backUpBurstTime, fileTable[i].responseTime);
    }
    printf("\n");
}

void printReadyQueueEntries(readyQueueNode *readyQueue)
{
    // Helper function to display ready queue entries, mainly for debugging and report
    readyQueueNode *temp = readyQueue;
    if (ITEMS_IN_QUEUE == 0)
    {
        printf("Queue currently empty\n");
    }
    printf("Ready Queue\tArrival Time\tBurst Time\tWaiting Time\tBackUp BurstTime\tTurnaround\tTime Added\tCompleted\tResponse time\n");
    while (temp != NULL)
    {
        if (temp->file.inQueue = true)
        {
            printf("\t\t%i\t\t%i\t\t%i\t\t%i\t\t\t%i\t\t%i\t\t%d\t\t%i\n", temp->file.arrivalTime, temp->file.burstTime, temp->file.waitingTime, temp->file.backUpBurstTime, temp->file.turnAroundTime, temp->file.timeAddedToQueue, temp->file.hasCompleted, temp->file.responseTime);
            temp = temp->next;
        }
    }
    printf("\n");
}

void UpdateWaitingTimes(readyQueueNode *readyQueue)
{
    if (ITEMS_IN_QUEUE == 0) {
        // guard clause, since readyqueue was malloc() if malloc successfully, readyqueue will never be null
        return;
    }
     // Update waiting times of processes in queue
    readyQueueNode *temp = readyQueue;
    while (temp != NULL)
    {
        if (temp->file.processing == false && temp->file.hasCompleted == false && temp->file.startedButNotComp == false)
        {
            temp->file.waitingTime = TIME_ELAPSED - temp->file.arrivalTime;
        }
        temp = temp->next;
    }
}

void HybridRoundRobin(int TIME_QUANTUM, readyQueueNode *readyQueue, fileAttribute fileTable[], fileAttribute completedFileTable[], int numberOfFiles, int *processCompleted, int *completedIndex)
{
    // Process files / processes in ready queue
    // Time quantum here is dynamic and changes according to remaining burst time of process
    // Employs dynamic time quantum in RR and SJF whenever possible
    fileAttribute updatedFileEntry;
    int timejump = 0;
    int newProcessAddedToQueue = 0;
    int uncompletedProcessesInQueue = CountUncompletedProcessesInQueue(readyQueue);
   
    if (uncompletedProcessesInQueue == 0)
    {
        TIME_ELAPSED++;
        return;
    }

    readyQueueNode *temp = readyQueue;
    readyQueueNode *newTemp = readyQueue;

    if (uncompletedProcessesInQueue == 1)
    {
        // Only 1 uncompleted process, start processing this file
        while (temp != NULL)
        {
            if (temp->file.hasCompleted == false)
            {
                break;
            }
            temp = temp->next;
        }

        temp->file.processing = true;
        // Selecting a new time quantum
        if (TIME_QUANTUM >= temp->file.backUpBurstTime)
        {
            timejump = temp->file.backUpBurstTime; // variable used to increment time elapsed by
        }

        else if (TIME_QUANTUM <= temp->file.backUpBurstTime)
        {
            timejump = DEFAULT_TIME_QUANTUM;
        }
        TIME_QUANTUM = timejump;
    }

    else if (uncompletedProcessesInQueue > 1)
    {
        // More than 1 process in queue obtain new burst time and process for scheduling 
        temp->file.processing = false;
        if(temp->file.saved == false){
            temp->file.timeStopped = TIME_ELAPSED;
            temp->file.startedButNotComp = true;
            temp->file.saved = true;
        }
        temp->file.calculated = false;
        int fid = getLowestRemainingBurstTime(readyQueue);
        while (newTemp != NULL)
        {
            if (fid == newTemp->file.fileID)
            {
                temp = newTemp;
                break;
            }
            newTemp = newTemp->next;
        }
        // Process the file with lowest remaining burst time instead
        temp->file.processing = true;
        //temp->file.hasStarted = true;
        timejump = temp->file.backUpBurstTime;
        TIME_QUANTUM = timejump;
    }
  
    for (int i = 0; i < timejump; i++)
    {
        // This for loop iteratively increments the time elapsed, allows us to add new files to our queue at 
        // each time interval i.e., maintains order in which files are 'ready' 
        if (temp->file.responseTime == -1 && temp->file.hasCompleted == false)
        {
            temp->file.responseTime = TIME_ELAPSED - temp->file.arrivalTime; 
        }
        temp->file.backUpBurstTime -= 1;
        if (temp->file.backUpBurstTime == 0 && temp->file.hasCompleted == false)
        {
            temp->file.hasCompleted = true;
            temp->file.processing = false;
            temp->file.turnAroundTime = temp->file.burstTime + temp->file.waitingTime;
            updatedFileEntry = temp->file;
            completedFileTable[*completedIndex] = updatedFileEntry;
            *processCompleted = *processCompleted + 1;
            *completedIndex = *completedIndex + 1;
            uncompletedProcessesInQueue = CountUncompletedProcessesInQueue(readyQueue);
        }
        else if(temp->file.backUpBurstTime > 0 && temp->file.calculated == false){
            temp->file.waitingTime += (TIME_ELAPSED - temp->file.timeStopped);
            temp->file.calculated = true;
            temp->file.saved = false;
            temp->file.startedButNotComp = false;
        }

        populateQueueIfArrive(numberOfFiles, fileTable, readyQueue);
        if(temp->file.startedButNotComp==false){
            UpdateWaitingTimes(readyQueue);
        }
        TIME_ELAPSED++;
    }
}
void populateQueueIfArrive(int numberOfFiles, fileAttribute fileTable[], readyQueueNode *readyQueue)
{
    // Populates ready queue by invoking AddToReadyQueue
    for (int i = 0; i < numberOfFiles; i++)
    {
        if (ITEMS_IN_QUEUE == numberOfFiles)
        {
            // Queue contains all files
            return;
        }
        else if (fileTable[i].inQueue == true)
        {
            // file already in queue, ignore
            continue;
        }

        else if (fileTable[i].hasCompleted == true)
        {
            // file already in queue and completed, ignore
            continue;
        }

        else if (TIME_ELAPSED > fileTable[i].arrivalTime && fileTable[i].inQueue == false)
        {
            // time has not elapsed for process to arrive ignore
            continue;
        }

        else if (fileTable[i].arrivalTime <= TIME_ELAPSED && fileTable[i].inQueue == false)
        {
            // File has arrived and is not yet in queue, i.e., add to queue 
            fileTable[i].hasArrived = true;
            fileTable[i].inQueue = true;
            fileTable[i].timeAddedToQueue = TIME_ELAPSED;
            AddToReadyQueue(fileTable[i], readyQueue);
        }
    }
}

void AddToReadyQueue(fileAttribute fileTableEntry, readyQueueNode *readyQueue)
{
    // Adds process to ready queue once the arrival time is lower or equals to time elapsed
    readyQueueNode *newNode = malloc(sizeof(readyQueueNode));
    newNode->file = fileTableEntry;
    newNode->next = NULL;
    
    // If queue initially empty, simply add file entry to head of list
    if (ITEMS_IN_QUEUE == 0)
    {
        readyQueue->file = newNode->file;
        readyQueue->next = NULL;
        readyQueue->file.inQueue = true;
        ITEMS_IN_QUEUE++;
        return;
    }

    // If queue not empty add to back of queue
    readyQueueNode *temp = readyQueue;
    while (temp->next != NULL)
    {
        temp = temp->next;
    }
    
    temp->next = malloc(sizeof(readyQueueNode));

    if (temp->next == NULL) {
        printf("Failed to allocate memory\n");
        exit(2);
    }

    temp->next->file = newNode->file;
    temp->next->next = NULL;
    readyQueue->file.inQueue = true;
    ITEMS_IN_QUEUE++;
    free(newNode);
    return;
}

//checking that input is integer
int validate(char *str)
{
    while(*str)
    {
        if(!isdigit(*str)){
            return 0; //return 0 if failure
        }
        str++; //proceed to next char in string
    }
    return 1; //return 1 if is digit
}


int getLowestRemainingBurstTime(readyQueueNode *readyQueue)
{
    // Obtain file ID of ready processes that has the lowest remaining burst time
    // if multiple process with same remaining burst times return fid of process with lowest arrival time (FIFO)
    readyQueueNode *temp = readyQueue;
    readyQueueNode *newTemp = readyQueue;

    int lowestBurstTime;
    int fid;
    int arrival;

    while (temp != NULL)
    {
        // Ignore if file has completed
        if (temp->file.hasCompleted == true)
        {
            temp = temp->next;
            continue;
        }
        // obtain fid of file burst time
        if (temp->file.backUpBurstTime > 0)
        {
            lowestBurstTime = temp->file.backUpBurstTime;
            fid = temp->file.fileID;
            arrival = temp->file.timeAddedToQueue;
        }
        temp = temp->next;
        break;
    }

    while (newTemp != NULL)
    {
        // find fid of file with lowest burst time
        if (lowestBurstTime > newTemp->file.backUpBurstTime && newTemp->file.hasCompleted == false)
        {
            lowestBurstTime = newTemp->file.backUpBurstTime;
            fid = newTemp->file.fileID;
        }

        // this allows a FIFO approach if two processes have same remaining burst times
        else if (lowestBurstTime == newTemp->file.backUpBurstTime)
        {
            if (arrival > newTemp->file.arrivalTime)
            {
                lowestBurstTime = newTemp->file.backUpBurstTime;
                fid = newTemp->file.fileID;
                arrival = newTemp->file.arrivalTime;
                break;
            }
        }
        newTemp = newTemp->next;
    }
    return fid;
}

int CountUncompletedProcessesInQueue(readyQueueNode *readyQueue)
{
    // returns uncompleted processes in queue i.e., hasCompleted flag not set (remaining burst time > 0)
    int uncompletedProcesses = 0;

    if (ITEMS_IN_QUEUE == 0) {
        return uncompletedProcesses;
    }

    readyQueueNode *temp = readyQueue;
    while (temp != NULL)
    {
        if (temp->file.hasCompleted == false)
        {
            uncompletedProcesses++;
        }
        temp = temp->next;
    }
    return uncompletedProcesses;
}

// Functions that return average or max/min burst and waiting times 
float GetAverageWaitingTime(fileAttribute completedFileTable[], int numberOfFiles) 
{
    float averageWaitingTime = 0.0;
    for (int i = 0; i < numberOfFiles; i++) {
        averageWaitingTime += completedFileTable[i].waitingTime;
    }
    averageWaitingTime = averageWaitingTime/numberOfFiles;
    printf("average waiting time: %.2f\n", averageWaitingTime);
    return averageWaitingTime;
}

float GetAverageTurnaroundTime(fileAttribute completedFileTable[], int numberOfFiles) 
{
    float averageTurnAroundTime = 0.0;
    for (int i = 0; i < numberOfFiles; i++){
        averageTurnAroundTime += completedFileTable[i].turnAroundTime;
    }
    averageTurnAroundTime = averageTurnAroundTime/numberOfFiles;
    printf("average turnaround time: %.2f\n", averageTurnAroundTime);
    return averageTurnAroundTime;
}

float GetMaxWaitingTime(fileAttribute completedFileTable[], int numberOfFiles) {
    float maxWaitTime = completedFileTable[0].waitingTime;
    for (int i = 1; i < numberOfFiles; i++){
        if (maxWaitTime < completedFileTable[i].waitingTime) {
            maxWaitTime = completedFileTable[i].waitingTime;
        }
    }
    printf("maximum waiting time: %.2f\n", maxWaitTime);
    return maxWaitTime;
}

float GetMinWaitingTime(fileAttribute completedFileTable[], int numberOfFiles) {
    float minWaitTime = completedFileTable[0].waitingTime;
    for (int i = 1; i < numberOfFiles; i++){
        if (minWaitTime > completedFileTable[i].waitingTime) {
            minWaitTime = completedFileTable[i].waitingTime;
        }
    }
    printf("minimum waiting time: %.2f\n", minWaitTime);
    return minWaitTime;
}

float GetMaxTurnAroundTime(fileAttribute completedFileTable[], int numberOfFiles) {
    float maxTurn = completedFileTable[0].turnAroundTime;
    for (int i = 1; i < numberOfFiles; i++){
        if (maxTurn < completedFileTable[i].turnAroundTime) {
            maxTurn = completedFileTable[i].turnAroundTime;
        }
    }
    printf("maximum turnaround time: %.2f\n", maxTurn);
    return maxTurn;
}

float GetMinTurnAroundTime(fileAttribute completedFileTable[], int numberOfFiles) {
    float minTurn = completedFileTable[0].turnAroundTime;
    for (int i = 1; i < numberOfFiles; i++){
        if (minTurn> completedFileTable[i].turnAroundTime) {
            minTurn= completedFileTable[i].turnAroundTime;
        }
    }
    printf("minimum turnaround time: %.2f\n", minTurn);
    return minTurn;
}

void printOrderOfCompletion(fileAttribute completedFileTable[], int numberOfFiles) {
    // prints sequence in which processes were completed
    printf("Order of completion: ");
    for (int i = 0; i < numberOfFiles; i++) {
        printf("[Process-%i]", completedFileTable[i].fileID + 1);
        if (i != numberOfFiles - 1) {
            printf(" -> ");
        }
    }
    printf("\n");
}

void freeAllocatedMemory(readyQueueNode *readyQueue) {
    // Frees memory allocated for linked list (ReadyQueue)
    readyQueueNode *tmp = NULL;
    while (readyQueue != NULL) {
        tmp = readyQueue;
        readyQueue = readyQueue->next;
        free(tmp);
    }
}
