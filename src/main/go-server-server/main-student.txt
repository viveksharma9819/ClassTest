/*
@Author: Vivek Sharma
*/

package main

import (
	"encoding/json"
	"fmt"
	"io/ioutil"
	"log"
	"mux-master"
	"net/http"
	"os"
	"regexp"
)

//Route export
type Route struct {
	Name        string
	Method      string
	Pattern     string
	HandlerFunc http.HandlerFunc
}

//Class struct
type Class struct {
	TotalStudents int     `json:"totalStudents,omitempty"`
	MaxMarks      float64 `json:"maxMarks,omitempty"`
	MinMarks      float64 `json:"minMarks,omitempty"`
	AvgMarks      float64 `json:"avgMarks,omitempty"`
}

//Student struct
type Student struct {
	ID     string  `json:"id,omitempty"`
	Name   string  `json:"name"`
	Age    int     `json:"age"`
	Marks  float64 `json:"marks"`
	Status string  `json:"status"`
}

//Students struct
type Students struct {
	Students []Student `json:"students"`
}

var students Students
var _students []Student

//GetAllStudentsDetails handler
func GetAllStudentsDetails(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(_students)
}

//AddStudent handler
func AddStudent(w http.ResponseWriter, r *http.Request) {

	var student Student
	_ = json.NewDecoder(r.Body).Decode(&student)

	//Validate Student before making an entry in the json file db
	if validateStudent(student, w) == false {
		return
	}

	// check for pass/fail status
	student.Status = "pass"
	if student.Marks < 65 {
		student.Status = "fail"
	}

	_students = append(_students, student)
	UpDateJSONStudentsDBFile()
	json.NewEncoder(w).Encode(student)

}

//GetStudentByID handler
func GetStudentByID(w http.ResponseWriter, r *http.Request) {

	params := mux.Vars(r)

	for _, item := range _students {

		if item.ID == params["id"] {
			json.NewEncoder(w).Encode(item)
			return
		}

	}
	json.NewEncoder(w).Encode(Student{})

}

//GetClassDetails handler
func GetClassDetails(w http.ResponseWriter, r *http.Request) {

	// return a default instance is there are not students yet added to the db
	if len(students.Students) == 0 {
		json.NewEncoder(w).Encode(Class{})
		return
	}

	var totalMarks float64
	var minMarks = 1000.0 // assuming a no which is greater or equal to max marks.
	var maxMarks = 0.0
	for _, value := range students.Students {

		if value.Marks < minMarks {
			minMarks = value.Marks
		}

		if value.Marks > maxMarks {
			maxMarks = value.Marks
		}

		totalMarks += value.Marks
	}

	var avgMarks = totalMarks / float64(len(students.Students))

	var _class Class
	_class.TotalStudents = len(students.Students)
	_class.MinMarks = minMarks
	_class.MaxMarks = maxMarks
	_class.AvgMarks = avgMarks

	json.NewEncoder(w).Encode(_class)

}

//DeleteStudent handler
func DeleteStudent(w http.ResponseWriter, r *http.Request) {

	params := mux.Vars(r)

	for index, item := range _students {

		if item.ID == params["id"] {
			_students = append(_students[:index], _students[index+1:]...)
			break
		}

	}
	UpDateJSONStudentsDBFile()
	json.NewEncoder(w).Encode(_students)

}

//UpdateStudent handler
func UpdateStudent(w http.ResponseWriter, r *http.Request) {
	params := mux.Vars(r)

	for index, item := range _students {

		if item.ID == params["id"] {
			_students = append(_students[:index], _students[index+1:]...)
			break
		}

	}
	UpDateJSONStudentsDBFile()

	var student Student
	_ = json.NewDecoder(r.Body).Decode(&student)

	//Validate Student before making an entry in the json file db
	if validateStudent(student, w) == false {
		return
	}

	// check for pass/fail status
	student.Status = "pass"
	if student.Marks < 65 {
		student.Status = "fail"
	}

	_students = append(_students, student)
	UpDateJSONStudentsDBFile()
	json.NewEncoder(w).Encode(student)
}

//GetDataFromJSONStudentsDBFile get students data and set it into students collection object
func GetDataFromJSONStudentsDBFile() {

	jsonFile, err := os.Open("studentsStore.json")
	// if  os.Open returns an error then handle it
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println("Successfully Opened studentsStore.json")
	// defer the closing of our jsonFile so that we can parse it later on

	byteValue, _ := ioutil.ReadAll(jsonFile)
	// we initialize our Users array

	// we unmarshal our byteArray which contains our
	// jsonFile's content into 'students' which we defined above
	json.Unmarshal(byteValue, &students)
	fmt.Println("un- marshalled")

	_students = students.Students
	fmt.Println("students collection initialized from json file db successfully..")
	defer jsonFile.Close()
}

//UpDateJSONStudentsDBFile students data
func UpDateJSONStudentsDBFile() {

	jsonFile, err := os.Open("studentsStore.json")
	// if we os.Open returns an error then handle it
	if err != nil {
		fmt.Println(err)
	}

	fmt.Println("Successfully Opened studentsStore.json")

	byteValue, _ := ioutil.ReadAll(jsonFile)

	// we initialize our students array
	// we unmarshal our byteArray which contains our
	// jsonFile's content into 'students' which we defined above
	json.Unmarshal(byteValue, &students)

	students.Students = _students
	_byteRR, err := json.Marshal(&students)
	_err := ioutil.WriteFile("studentsStore.json", _byteRR, 0644)
	if _err != nil {
		log.Fatal(err)
	}
	_students = students.Students

	fmt.Println("students collection updated from json file db")
	defer jsonFile.Close()
}

//validation helper method
func validateStudent(student Student, w http.ResponseWriter) bool {
	name := student.Name

	// using negation check to identify name has special chars --except space between names say first name middle name last name
	reg, err := regexp.Compile("[^a-zA-Z' ']+")
	if err != nil {
		log.Fatal(err)
	}

	if len(student.Name) > 255 {
		json.NewEncoder(w).Encode("Validation !!! Name is not allowed to have more than 255 characters")
		return false
	}

	processedString := reg.ReplaceAllString(name, "")
	if processedString != student.Name {
		json.NewEncoder(w).Encode("Validation !!! Name can not contain special characters except space")
		return false
	}

	if student.Age > 100 {
		json.NewEncoder(w).Encode("Validation !!!  Age can not be greater than 100")
		return false
	}

	return true
}

func main() {
	port := ":1500"
	router := mux.NewRouter()
	router.HandleFunc("/students/{id}", UpdateStudent).Methods("PUT")
	router.HandleFunc("/students/{id}", GetStudentByID).Methods("GET")
	router.HandleFunc("/students", GetAllStudentsDetails).Methods("GET")
	router.HandleFunc("/classDetails", GetClassDetails).Methods("GET")
	router.HandleFunc("/students/{id}", AddStudent).Methods("POST")
	router.HandleFunc("/students/{id}", DeleteStudent).Methods("DELETE")
	// set students collection from json file db
	GetDataFromJSONStudentsDBFile()
	fmt.Println("Listening on port:", port)
	log.Fatal(http.ListenAndServe(port, router))

}
