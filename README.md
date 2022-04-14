## Please write pseudocode employee time off request feature:

Form request

Time Off Type

Date

Reason

Upload file (optional)

Save data to database

Send email to the employee’s manager



# 1. Define Struct
```
type TimeOffRequest struct {
	EmployeeID  int       `validate:"required"`
	TimeOffType string    `validate:"required"`
	Date        time.Time `validate:"required"`
	Reason      string    `validate:"required"`
	File        string    `validate:"omitempty"`
}
```

# 2. Define repo & usecase interface
```
type TimeOffRequestRepository interface {
  Create(TimeOffRequest) (TimeOffRequest, error)
}
type TimeOffRequestUsecase interface {
  Create(TimeOffRequest) (TimeOffRequest, error)
}
```

# 3 create queue for sending email to manager
```
func (q Queue) sendEmail(timeOffRequest TimeOffRequest) {

  managerEmail := os.Getenv("MANAGER_EMAIL")
  
  err := q.redisClient.RPush("email-queue", managerEmail).Err()
  if err != nil {
    log.Printf("Error sending email to manager: %v", err)
  }
}
```

# 4. worker for sending email to manager
```
func main() {
  redisClient := redis.NewClient(&redis.Options{
    Addr: "localhost:6379",
  })

  for {
    email, err := redisClient.BLPop(0, "email-queue").Result()
    if err != nil {
      log.Fatal(err)
    }

    managerEmail := email[1]
    // send email to manager
    fmt.Println("send email to manager:", managerEmail)
  }
}
```

# 5. implement repo interface
```
type TimeOffRequestRepositoryImpl struct {
  db *sql.DB
}

func (r *TimeOffRequestRepositoryImpl) Create(request TimeOffRequest) (TimeOffRequest, error) {

  query := `
    INSERT INTO time_off_request (employee_id, time_off_type, date, reason, file)
    VALUES (?, ?, ?, ?, ?)
  `

  stmt, err := r.db.Prepare(query)

  if err != nil {
    return request, err
  }

  defer stmt.Close()

  _, err = stmt.Exec(request.EmployeeID, request.TimeOffType, request.Date, request.Reason, request.File)

  if err != nil {
    return request, err
  }

  // send email to manager
  sendEmail(request)

  return request, nil
}
```

# 6. implement usecase interface
```
type TimeOffRequestUsecaseImpl struct {
  repo TimeOffRequestRepository
}

func (u *TimeOffRequestUsecaseImpl) Create(request TimeOffRequest) (TimeOffRequest, error) {
  return u.repo.Create(request)
}
```

# 7. implement http handler
```
type TimeOffRequestHandler struct {
  usecase TimeOffRequestUsecase
}

// @Description Create Time Off Request
// @Accept  json
// @Produce  json
// @Param request body TimeOffRequest true "Create Time Off Request"
// @Success 201 {object} TimeOffRequest
// @Failure 400 {object} TimeOffRequest
// @Failure 500 {object} TimeOffRequest
// @Router /time-off-request [post]
func (h *TimeOffRequestHandler) Create(w http.ResponseWriter, r *http.Request) {
  var request TimeOffRequest

  err := json.NewDecoder(r.Body).Decode(&request)

  if err != nil {
    http.Error(w, err.Error(), http.StatusBadRequest)
    return
  }

  _, err = h.usecase.Create(request)

  if err != nil {
    http.Error(w, err.Error(), http.StatusInternalServerError)
    return
  }

  w.WriteHeader(http.StatusCreated)
}
```




