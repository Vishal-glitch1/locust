import math
from locust import HttpUser, task, constant, SequentialTaskSet, LoadTestShape, events, between


class UserTasks(SequentialTaskSet):


    @task
    def login(self):
        user_id = "user_id"  # Placeholder for a user ID
        # This line sends a POST request to the server. It sends a JSON object (which is a standard format for sending data over the web) with the user ID inside. In this case, the user ID is set to the placeholder value "user_id".
        self.client.post("/login", json={"user_id": user_id})


    @task
    def create_order(self):
        self.client.post("/order", json={})


    @task
    def make_payment(self):
        order_id = 1  # Use a valid order ID
        self.client.post("/payment", json={"order_id": order_id})

class WebsiteUser(HttpUser):
    wait_time = constant(1)
    tasks = [UserTasks]

class StepLoadShape(LoadTestShape):
    step_time = 10
    step_load = 50
    spawn_rate = 50
    time_limit = 600 #Not needed in current scenario as another stop criteria is defined based on RT

    def tick(self):
        run_time = self.get_run_time()

        # Stop the test if the run time exceeds the time limit
        if run_time > self.time_limit:
            return None

        # Check average response time for the payment endpoint
        avg_response_time = self.get_average_response_time()
        if avg_response_time is not None:
            print(f"Current average response time for payment: {avg_response_time:.2f} ms")
            if avg_response_time > 100:
                print("Average response time exceeded 100 ms. Stopping the test.")
                return None  # Stop the test

        current_step = math.floor(run_time / self.step_time)
        return (current_step * self.step_load, self.spawn_rate)

    def get_average_response_time(self):
        """Calculate average response time for the payment endpoint."""
        # Specify the HTTP method when getting stats
        payment_stats = self.environment.stats.get('/payment', 'POST')  # Include the method
       # print(f"Environment: {payment_stats} ")
       # print(vars(payment_stats))
        if payment_stats:
            return payment_stats.avg_response_time  # Return the average response time (Total_response_time/num_request)
        return None

# Register the load shape with the environment
@events.init.add_listener #This decorator is used to register a listener for the init event in Locust. The init event is triggered when the Locust environment is initialized, allowing you to run your custom initialization code at that time.
def on_locust_init(environment, **_kwargs):
    StepLoadShape.environment = environment  # Store the environment in the class
    environment.shape_class = StepLoadShape()  # Assign the shape class
