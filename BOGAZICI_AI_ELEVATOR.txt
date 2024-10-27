#include <vector>
#include <queue>
#include <cmath>
#include <algorithm>
#include <memory>
#include <chrono>
#include <map>

// Forward declarations
class Passenger;
class ElevatorSystem;

// Passenger state
enum class PassengerState {
    WAITING,
    IN_ELEVATOR,
    DELIVERED
};

// Direction of travel
enum class Direction {
    UP,
    DOWN,
    IDLE
};

// Time-based passenger frequency model
class PassengerFrequencyModel {
private:
    // Peak hours (morning: 8-10, evening: 17-19)
    const std::vector<std::pair<int, int>> peakHours = {{8,10}, {17,19}};
    const double baseFrequency;
    const double peakMultiplier;

public:
    PassengerFrequencyModel(double baseFreq = 0.1, double peakMult = 3.0)
        : baseFrequency(baseFreq), peakMultiplier(peakMult) {}

    double getFrequency(int hour) const {
        for (const auto& peak : peakHours) {
            if (hour >= peak.first && hour < peak.second) {
                return baseFrequency * peakMultiplier;
            }
        }
        return baseFrequency;
    }
};

// Passenger class to track individual passengers
class Passenger {
public:
    int fromFloor;
    int toFloor;
    int startTime;
    PassengerState state;
    int elevatorId;

    Passenger(int from, int to, int time)
        : fromFloor(from), toFloor(to), startTime(time),
          state(PassengerState::WAITING), elevatorId(-1) {}

    Direction getDirection() const {
        if (toFloor > fromFloor) return Direction::UP;
        return Direction::DOWN;
    }
};

// Enhanced Elevator class
class Elevator {
private:
    static constexpr int DOOR_OPERATION_TIME = 3;
    static constexpr int ACCELERATION_ENERGY = 2;
    static constexpr double MAX_LOAD = 0.8;  // 80% of capacity

    int currentFloor;
    Direction direction;
    std::vector<std::shared_ptr<Passenger>> passengers;
    std::map<int, bool> stopRequests;
    int capacity;
    int doorTimer;
    bool doorsOpen;
    double energyConsumption;
    
public:
    Elevator(int startFloor, int cap = 10)
        : currentFloor(startFloor), direction(Direction::IDLE),
          capacity(cap), doorTimer(0), doorsOpen(false),
          energyConsumption(0) {}

    bool canAddPassenger() const {
        return passengers.size() < capacity * MAX_LOAD;
    }

    void addStopRequest(int floor) {
        stopRequests[floor] = true;
    }

    double getEnergyConsumption() const { return energyConsumption; }

    int getCurrentFloor() const { return currentFloor; }
    Direction getDirection() const { return direction; }

    void addPassenger(std::shared_ptr<Passenger> passenger) {
        if (canAddPassenger()) {
            passengers.push_back(passenger);
            passenger->state = PassengerState::IN_ELEVATOR;
            addStopRequest(passenger->toFloor);
        }
    }

    // Main update function
    void update() {
        if (doorsOpen) {
            handleDoors();
            return;
        }

        if (shouldStop()) {
            startDoorOperation();
            return;
        }

        move();
    }

private:
    void handleDoors() {
        doorTimer--;
        if (doorTimer <= 0) {
            doorsOpen = false;
            energyConsumption += DOOR_OPERATION_TIME;
            
            // Remove delivered passengers
            passengers.erase(
                std::remove_if(passengers.begin(), passengers.end(),
                    [this](const std::shared_ptr<Passenger>& p) {
                        if (p->toFloor == currentFloor) {
                            p->state = PassengerState::DELIVERED;
                            return true;
                        }
                        return false;
                    }
                ),
                passengers.end()
            );

            stopRequests.erase(currentFloor);
            updateDirection();
        }
    }

    void startDoorOperation() {
        doorsOpen = true;
        doorTimer = DOOR_OPERATION_TIME;
        energyConsumption += DOOR_OPERATION_TIME;
    }

    bool shouldStop() const {
        return stopRequests.count(currentFloor) > 0;
    }

    void move() {
        if (direction == Direction::IDLE) return;

        // Add acceleration energy cost when starting movement
        if (!wasMoving) {
            energyConsumption += ACCELERATION_ENERGY;
        }

        if (direction == Direction::UP) {
            currentFloor++;
        } else if (direction == Direction::DOWN) {
            currentFloor--;
        }

        // Basic energy consumption for movement
        energyConsumption += 1.0 + (0.1 * passengers.size());
    }

    void updateDirection() {
        if (stopRequests.empty()) {
            direction = Direction::IDLE;
            return;
        }

        // Keep going in same direction if there are requests
        if (direction == Direction::UP) {
            auto it = stopRequests.upper_bound(currentFloor);
            if (it == stopRequests.end()) {
                direction = Direction::DOWN;
            }
        } else if (direction == Direction::DOWN) {
            auto it = stopRequests.lower_bound(currentFloor);
            if (it == stopRequests.begin()) {
                direction = Direction::UP;
            }
        } else {
            // If idle, choose direction based on closest request
            auto it = stopRequests.begin();
            direction = (it->first > currentFloor) ? Direction::UP : Direction::DOWN;
        }
    }
};

// Improved Elevator System
class ElevatorSystem {
private:
    std::vector<Elevator> elevators;
    std::vector<std::shared_ptr<Passenger>> waitingPassengers;
    PassengerFrequencyModel frequencyModel;
    
    // Statistics
    double totalEnergyConsumed;
    double totalWaitingTime;
    int numDeliveredPassengers;
    std::map<int, std::vector<double>> waitTimesByHour;

public:
    ElevatorSystem(int numElevators, int startingFloor)
        : totalEnergyConsumed(0), totalWaitingTime(0), numDeliveredPassengers(0) {
        for (int i = 0; i < numElevators; ++i) {
            elevators.emplace_back(startingFloor);
        }
    }

    void newCall(int from, int to, int currentTime) {
        auto passenger = std::make_shared<Passenger>(from, to, currentTime);
        waitingPassengers.push_back(passenger);
        assignElevator(passenger);
    }

    void update(int currentTime) {
        // Update elevators
        for (auto& elevator : elevators) {
            elevator.update();
            totalEnergyConsumed += elevator.getEnergyConsumption();
        }

        // Update waiting times and assign unassigned passengers
        for (auto& passenger : waitingPassengers) {
            if (passenger->state == PassengerState::WAITING) {
                totalWaitingTime += 1;
                assignElevator(passenger);
            }
        }

        // Clean up delivered passengers and update statistics
        updateStatistics(currentTime);
    }

    void runSimulation(int numFloors, int totalHours) {
        int timeStep = 0;
        int currentHour = 0;

        while (currentHour < totalHours) {
            timeStep++;
            if (timeStep % 3600 == 0) currentHour++;  // 3600 steps = 1 hour

            // Generate new passengers based on time-dependent frequency
            double freq = frequencyModel.getFrequency(currentHour % 24);
            if (static_cast<double>(rand()) / RAND_MAX < freq) {
                int from = rand() % numFloors;
                int to;
                // Realistic floor selection based on time of day
                if (currentHour % 24 < 12) {
                    // Morning: more likely to go up
                    to = from + (rand() % (numFloors - from));
                } else {
                    // Evening: more likely to go down
                    to = rand() % (from + 1);
                }
                if (from != to) {
                    newCall(from, to, timeStep);
                }
            }

            update(timeStep);
        }
        printStatistics();
    }

private:
    void assignElevator(std::shared_ptr<Passenger> passenger) {
        int bestElevatorIndex = -1;
        double bestScore = std::numeric_limits<double>::max();

        for (int i = 0; i < elevators.size(); ++i) {
            Elevator& elevator = elevators[i];
            
            if (!elevator.canAddPassenger()) continue;

            // Calculate score based on multiple factors
            double score = calculateAssignmentScore(elevator, passenger);
            
            if (score < bestScore) {
                bestScore = score;
                bestElevatorIndex = i;
            }
        }

        if (bestElevatorIndex != -1) {
            elevators[bestElevatorIndex].addPassenger(passenger);
            passenger->elevatorId = bestElevatorIndex;
        }
    }

    double calculateAssignmentScore(const Elevator& elevator, const std::shared_ptr<Passenger>& passenger) {
        double distance = std::abs(elevator.getCurrentFloor() - passenger->fromFloor);
        double directionalPreference = 1.0;

        // Prefer elevators already moving in the right direction
        if (elevator.getDirection() != Direction::IDLE) {
            if (elevator.getDirection() == passenger->getDirection()) {
                directionalPreference = 0.5;
            } else {
                directionalPreference = 2.0;
            }
        }

        // Consider energy efficiency
        double energyFactor = distance * 0.5;  // Less weight on energy compared to waiting time

        return (distance + energyFactor) * directionalPreference;
    }

    void updateStatistics(int currentTime) {
        waitingPassengers.erase(
            std::remove_if(waitingPassengers.begin(), waitingPassengers.end(),
                [this, currentTime](const std::shared_ptr<Passenger>& p) {
                    if (p->state == PassengerState::DELIVERED) {
                        int waitTime = currentTime - p->startTime;
                        int hour = (currentTime / 3600) % 24;
                        waitTimesByHour[hour].push_back(waitTime);
                        numDeliveredPassengers++;
                        return true;
                    }
                    return false;
                }
            ),
            waitingPassengers.end()
        );
    }

    void printStatistics() const {
        std::cout << "\nSimulation Results:\n";
        std::cout << "Total Energy Consumed: " << totalEnergyConsumed << " units\n";
        std::cout << "Average Waiting Time: " << (numDeliveredPassengers > 0 ? 
            totalWaitingTime / numDeliveredPassengers : 0) << " seconds\n";
        
        // Print waiting times by hour
        std::cout << "\nAverage Waiting Times by Hour:\n";
        for (const auto& [hour, times] : waitTimesByHour) {
            if (!times.empty()) {
                double avg = std::accumulate(times.begin(), times.end(), 0.0) / times.size();
                std::cout << "Hour " << hour << ": " << avg << " seconds\n";
            }
        }
    }
};
