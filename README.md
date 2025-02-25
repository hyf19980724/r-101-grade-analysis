# Grade analysis lab

The goal of this lab is to analyse a real world collection of marks
(grades) obtained by students during a semester. 

### Question 1:

```{r}
library(here)
library(vroom)
library(dplyr)
library(tidyr)
library(ggplot2)
here::i_am("r-101-grade-analysis.Rproj")
grades <- vroom(here("grades.csv"))
```

### Question 2:

```{r}
exam_stats <- grades %>%
  summarise(
    Min = min(Exam, na.rm = TRUE),
    Max = max(Exam, na.rm = TRUE),
    Median = median(Exam, na.rm = TRUE),
    Mean = mean(Exam, na.rm = TRUE)
  )
knitr::kable(exam_stats, caption = "Descriptive statistics of the Exam grades")
```

### Question 3:

```{r}
num_missing_exam <- grades %>%
  summarise(Count = sum(is.na(Exam)))
missing_count <- num_missing_exam$Count
cat("The number of students who did not take the final exam is", missing_count, ".")
```

### Question 4:

```{r}
grades %>%
  filter(!is.na(Exam)) %>%
  ggplot(aes(x = Exam)) +
  geom_histogram(binwidth = 1, fill = "blue", color = "black", alpha = 0.7) + 
  labs(title = "Distribution of Exam Grades", x = "Grades", y = "Number of Students") +
  theme_minimal()
```

### Question 5:

```{r}
students_per_group <- grades %>%
  group_by(Group) %>%
  summarise(Count = n())
knitr::kable(students_per_group, caption = "Number of Students in Each Group")
```

### Question 6:

```{r}
ggplot(students_per_group, aes(x = Group, y = Count)) +
  geom_bar(stat = "identity", fill = "steelblue") +
  labs(title = "Number of Students in Each Group", x = "Group", y = "Number of Students")+
  theme(axis.text.x = element_text(angle = 90, hjust = 1, vjust = 0.5))
```

### Question 7:

```{r}
grades %>%
  filter(!is.na(Exam)) %>%
  ggplot(aes(x = Group, y = Exam, fill = Group)) +
  geom_boxplot() +
  labs(title = "Distribution of Exam Grades by Group", x = "Group", y = "Exam Grade") +
  theme_minimal() +
  theme(legend.position = "none")+
  theme(axis.text.x = element_text(angle = 90, hjust = 1, vjust = 0.5))
```

### Question 8:

```{r}
students_missed_exam <- grades %>%
  group_by(Group) %>%
  summarise(Missed_Exam = sum(is.na(Exam)))
knitr::kable(students_missed_exam, caption = "Number of Students Who Missed the Exam in Each Group")
```

### Question 9:

```{r}
ggplot(students_missed_exam, aes(x = Group, y = Missed_Exam)) +
  geom_col(fill = "red") +  
  labs(title = "Number of Students Who Missed the Exam per Group", 
       x = "Group", 
       y = "Number of Students") +
  theme_minimal() +
  scale_y_continuous(breaks = seq(0, max(students_missed_exam$Missed_Exam, na.rm = TRUE), by = 1))+
  theme(axis.text.x = element_text(angle = 90, hjust = 1, vjust = 0.5))
```

### Question 10:

```{r}
long_format <- grades %>%
  tidyr::pivot_longer(
    cols = -c(Id, Group), 
    names_to = "name",  
    values_to = "Grade" )
knitr::kable(long_format, caption = "Grades of Students in Each Question")
```

### Question 11:

```{r}
missing_grades_per_student <- long_format %>%
  group_by(Id) %>%
  summarise(Missing_Grades = sum(is.na(Grade)))
knitr::kable(missing_grades_per_student, caption = "Number of Missing Grades Per Student")
```

### Question 12:

```{r}
ggplot(missing_grades_per_student, aes(x = Missing_Grades)) +
  geom_histogram(binwidth = 1, fill = 'skyblue', color = 'black') +
  labs(title = "Distribution of Missing Grades for Per Student", x = "Number of Missing Grades", y = "Number of Students") +
  theme_minimal()
```

### Question 13:

```{r}
students_missed_exam_long <- long_format %>%
  filter(name == 'Exam' & is.na(Grade)) %>%  
  group_by(Group) %>%
  summarise(Missed_Exam = n()) %>%
  complete(Group, fill = list(Missed_Exam = 0))
knitr::kable(students_missed_exam_long, caption = "Number of Students Who Missed the Exam in Each Group")
```

### Question 14:

```{r}
library(stringr)
missing_online_grades <- long_format %>%
  filter(str_starts(name, 'Online_MCQ')) %>%
  group_by(Id) %>%
  summarise(missing_online_grades = sum(is.na(Grade)))
knitr::kable(missing_online_grades, caption = "Number of Missing Online Grades Per Student")
```

### Question 15:

```{r}
complete_data <- grades %>%
  left_join(missing_online_grades, by = "Id") %>%
  replace_na(list(missing_online_grades = 0))


ggplot(complete_data, aes(x = missing_online_grades, y = Exam, color = as.factor(missing_online_grades))) +
  geom_point(position = "jitter", na.rm = TRUE) +
  labs(title = "Exam Grades Based on the Number of Missing Online Grades", 
       x = "Number of Missing Online Tests", 
       y = "Exam Grade",
       color = "Missing Online Tests") 
```

### Question 16:

```{r}
students_missed_mcq <- grades %>%
  rowwise() %>%
  mutate(Missed = any(is.na(c_across(starts_with("MCQ_"))))) %>%
  select(Id, Missed)
knitr::kable(students_missed_mcq, caption = "Students Missing At Least One Grade")
```

### Question 17

```{r}
group_missed_percentage <- students_missed_mcq %>%
  left_join(grades %>% select(Id, Group), by = "Id") %>%
  group_by(Group) %>%
  summarise(P_missed = mean(Missed) * 100)
knitr::kable(group_missed_percentage, caption = "Percentage of Students Missing At Least One Grade by Group")
```

### Question 18:

```{r}
avg_exam_grade <- grades %>%
  group_by(Group) %>%
  summarise(Avg_Exam_Grade = mean(Exam, na.rm = TRUE))

exam_vs_missed_mcq <- inner_join(avg_exam_grade, group_missed_percentage, by = "Group")


ggplot(exam_vs_missed_mcq, aes(x = P_missed, y = Avg_Exam_Grade)) +
  geom_point() +  # Use geom_point() to create a scatterplot
  geom_smooth(method = 'lm', se = FALSE) +  # Add a linear regression line; set se = FALSE to remove the shaded confidence region
  labs(title = "Average Exam Grade vs. Percentage of Missed MCQs", 
       x = "Percentage of Missed MCQs", 
       y = "Average Exam Grade")
```
