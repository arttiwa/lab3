# lab3
package entity

import (
	"time"

	"gorm.io/gorm"
)

type User struct {
	gorm.Model
	FirstName            string `valid:"required~กรุณากรอกชื่อ"`
	LastName             string `valid:"required~กรุณากรอกนามสกุล"`
	Email                string `gorm:"uniqueIndex" valid:"email, required~Email: กรุณากรอกอีเมล"`
	PhoneNumber          string `gorm:"uniqueIndex" valid:"matches(^\\d{10}$)~เบอร์โทรศัพท์ต้องมีตัวเลข 10 หลัก, required~กรุณากรอกเบอร์โทรศัพท์"`
}
type Role struct {
	gorm.Model
	Name string `gorm:"uniqueIndex"`
	User []User `gorm:"foreignkey:RoleID"`
}



package entity

import (
	"testing"
	"time"

	"github.com/asaskevich/govalidator"
	. "github.com/onsi/gomega"
)
func TestUserPass(t *testing.T) {
	g := NewGomegaWithT(t)

	// ข้อมูลถูกต้องหมดทุก field
	user := User{
		FirstName:            "bbb",
		LastName:             "ccc",
		Email:                "bbb@gmail.com",
		PhoneNumber:          "1234567890",
		IdentificationNumber: "1234567890123",
		StudentID:            "B1234567",
		Age:                  20,
		Password:             "123456",
		BirthDay:             time.Now().AddDate(2002, 04, 20),
	}

	// ตรวจสอบด้วย govalidator
	ok, err := govalidator.ValidateStruct(user)

	// ok ต้องเป็น true แปลว่าไม่มี error
	g.Expect(ok).To(BeTrue())

	// err ต้องเป็น nil แปลว่าไม่มี error
	g.Expect(err).To(BeNil())
}
func TestUserPhonenumberNull(t *testing.T) {
	g := NewGomegaWithT(t)

	// ข้อมูล ไม่ถูกต้องตาม Format
	user := User{
		FirstName:            "bbb",
		LastName:             "ccc",
		Email:                "bbb@gmail.com",
		PhoneNumber:          "",
		IdentificationNumber: "1234567890123",
		StudentID:            "B6245854",
		Age:                  20,
		Password:             "123456",
		BirthDay:             time.Now().AddDate(2002, 04, 20),
	}

	// ตรวจสอบด้วย govalidator
	ok, err := govalidator.ValidateStruct(user)

	// ok ต้องไม่เป็น true แปลว่าต้องจับ error ได้
	g.Expect(ok).ToNot(BeTrue())

	// err ต้องไม่เป็น nil แปลว่าต้องจับ error ได้
	g.Expect(err).ToNot(BeNil())

	// err.Error() ต้องมี message แสดงออกมา
	g.Expect(err.Error()).To(Equal("กรุณากรอกเบอร์โทรศัพท์"))
}



package controller

import (
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/sut65/team14/entity"
)
func GetUser(c *gin.Context) {
	var user entity.User
	id := c.Param("id")
	if err := entity.DB().Preload("Role").Preload("Gender").Preload("EducationLevel").Raw("SELECT * FROM users WHERE id = ?", id).Find(&user).Error; err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusOK, gin.H{"data": user})
}

// GET /Users
func ListUsers(c *gin.Context) {
	var users []entity.User
	if err := entity.DB().Preload("Gender").Preload("Role").Preload("EducationLevel").Raw("SELECT * FROM users").Find(&users).Error; err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusOK, gin.H{"data": users})
}

// DELETE /users/:id
func DeleteUser(c *gin.Context) {
	id := c.Param("id")
	if tx := entity.DB().Exec("DELETE FROM users WHERE id = ?", id); tx.RowsAffected == 0 {
		c.JSON(http.StatusBadRequest, gin.H{"error": "user not found"})
		return
	}
	c.JSON(http.StatusOK, gin.H{"data": id})
}

// PATCH /users
func UpdateUser(c *gin.Context) {
	var user entity.User
	if err := c.ShouldBindJSON(&user); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	if tx := entity.DB().Where("id = ?", user.ID).First(&user); tx.RowsAffected == 0 {
		c.JSON(http.StatusBadRequest, gin.H{"error": "user not found"})
		return
	}
	if err := entity.DB().Save(&user).Error; err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusOK, gin.H{"data": user})
}

// GET /Genders
func ListGenders(c *gin.Context) {
	var genders []entity.Gender
	if err := entity.DB().Raw("SELECT * FROM genders").Scan(&genders).Error; err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusOK, gin.H{"data": genders})
}

// GET /EducationLevels
func ListEducationLevels(c *gin.Context) {
	var educationlevels []entity.EducationLevel
	if err := entity.DB().Raw("SELECT * FROM education_levels").Scan(&educationlevels).Error; err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusOK, gin.H{"data": educationlevels})
}

// GET /Roles
func ListRoles(c *gin.Context) {
	var roles []entity.Role
	if err := entity.DB().Raw("SELECT * FROM roles").Scan(&roles).Error; err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusOK, gin.H{"data": roles})
}

// GET /User/StudentID/:StudentID ---------- add friend ----------------
func GetUserByStudentID(c *gin.Context) {
	var user entity.User
	studentID := c.Param("StudentID")
	if err := entity.DB().Preload("Role").Preload("Gender").Preload("EducationLevel").Raw("SELECT * FROM users WHERE student_id = ?", studentID).Find(&user).Error; err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusOK, gin.H{"data": user})
}
