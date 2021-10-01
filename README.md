# PingPong

OPENGL utilizando GLFW e GLAD.


#include <glad/glad.h>
#include <GLFW/glfw3.h>



#include <string>
#include <iostream>
#include <sstream>
#include <fstream>

unsigned int scrWidth = 800;
unsigned int scrHeight = 600;
const char* title = "PingPong";
GLuint shaderProgram;

const float paddleSpeed = 75.0f;
const float paddleHeight = 100.0f;
const float halfPaddleHeight = paddleHeight / 2.0f;
const float paddleWidth = 10.0f;
const float halfPaddleWidth = paddleWidth / 2.0f;
const float ballDiameter = 16.0f;
const float ballRadius = ballDiameter / 2.0f;
const float offset = ballRadius;
const float paddleBoundary = halfPaddleHeight + offset;

struct vec2 {
	float x;
	float y;
};


vec2 ballOffset;
vec2 paddleOffsets[1];
float paddleVelocities[2];
vec2 initBallVelocity = { 300.0f, 300.0f };
vec2 ballVelocity = { 300.0f, 300.0f };

void initGLFW(unsigned int versionMajor, unsigned int versionMinor) {
	glfwInit();


	glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, versionMajor);
	glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, versionMinor);
	glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

#ifdef __APPLE__
	glfwWindowHint(GLFW_OPENGL_FORWARD_COMPACT, GL_TRUE);
#endif
}

void createWindow(GLFWwindow*& window,
	const char* title, unsigned int width, unsigned int height,
	GLFWframebuffersizefun framebufferSizeCallback) {
	window = glfwCreateWindow(width, height, title, NULL, NULL);
	if (!window) {
		return;
	}
	glfwMakeContextCurrent(window);
	glfwSetFramebufferSizeCallback(window, framebufferSizeCallback);

}



bool loadGlad() {
	return gladLoadGLLoader((GLADloadproc)glfwGetProcAddress);
}

std::string readFile(const char* filename) {
	std::ifstream file;
	std::stringstream buf;

	std::string ret = "";

	file.open(filename);

	if (file.is_open()) {
		buf << file.rdbuf();
		ret = buf.str();
	}
	else {
		std::cout << "Não foi possivel abrir" << filename << std::endl;
	}

	file.close();

	return ret;
}

int genShader(const char* filepath, GLenum type) {
	std::string shaderSrc = readFile(filepath);
	const GLchar* shader = shaderSrc.c_str();

	int shaderObj = glCreateShader(type);
	glShaderSource(shaderObj, 1, &shader, NULL);
	glCompileShader(shaderObj);

	int success;
	char infoLog[512];
	glGetShaderiv(shaderObj, GL_COMPILE_STATUS, &success);
	if (!success) {
		glGetShaderInfoLog(shaderObj, 512, NULL, infoLog);
		std::cout << "ERRO DE COMPILAÇÃO DO SHADER" << infoLog << std::endl;
		return -1;
	}

	return shaderObj;
}


int genShaderProgram(const char* vertexShaderPath, const char* fragmentShaderPath) {
	int shaderProgram = glCreateProgram();

	int vertexShader = genShader(vertexShaderPath, GL_VERTEX_SHADER);
	int fragmentShader = genShader(fragmentShaderPath, GL_FRAGMENT_SHADER);

	if (vertexShader == -1 || fragmentShader == -1) {
		return -1;
	}

	glAttachShader(shaderProgram, vertexShader);
	glAttachShader(shaderProgram, fragmentShader);
	glLinkProgram(shaderProgram);

	int success;
	char infoLog[512];
	glGetShaderiv(shaderProgram, GL_LINK_STATUS, &success);
	if (!success) {
		glGetShaderInfoLog(shaderProgram, 512, NULL, infoLog);
		std::cout << "ERRO NO SHADER" << infoLog << std::endl;
		return -1;
	}

	glDeleteShader(vertexShader);
	glDeleteShader(fragmentShader);

	return shaderProgram;
}


void bindShader(int shaderProgram) {
	glUseProgram(shaderProgram);
}

void setOrthographicProjection(int shaderProgram,
	float left, float right,
	float bottom, float top,
	float near, float far) {
	float mat[4][4] = {
		{ 2.0f / (right - left), 0.0f, 0.0f, 0.0f },
		{ 0.0f, 2.0f / (top - bottom), 0.0f, 0.0f },
		{ 0.0f, 0.0f, -2.0f / (far - near), 0.0f },
		{ -(right + left) / (right - left), -(top + bottom) / (top - bottom), -(far + near) / (far - near), 1.0f }
	};

	bindShader(shaderProgram);
	glUniformMatrix4fv(glGetUniformLocation(shaderProgram, "projection"), 1, GL_FALSE, &mat[0][0]);
	
}

void deleteShader(int shaderProgram) {
	glDeleteProgram(shaderProgram);
}

struct VAO {
	GLuint val;
	GLuint offsetVBO;
	GLuint posVBO;
	GLuint sizeVBO;
	GLuint EBO;
	
};

void genVAO(VAO* vao) {
	glGenVertexArrays(1, &vao->val);
	glBindVertexArray(vao->val);
}


template<typename T>
void genBufferObject(GLuint& bo, GLenum type, GLuint noElements, T* data, GLenum usage) {
	glGenBuffers(1, &bo);
	glBindBuffer(type, bo);
	glBufferData(type, noElements * sizeof(T), data, usage);
}

template<typename T>
void updateData(GLuint& bo, GLintptr offset, GLuint noElements, T* data) {
	glBindBuffer(GL_ARRAY_BUFFER, bo);
	glBufferSubData(GL_ARRAY_BUFFER, offset, noElements * sizeof(T), data);
}

template<typename T>
void setAttPointer(GLuint& bo, GLuint idx, GLint size, GLenum type, GLuint stride, GLuint offset, GLuint divisor = 0) {
	glBindBuffer(GL_ARRAY_BUFFER, bo);
	glVertexAttribPointer(idx, size, type, GL_FALSE, stride * sizeof(T), (void*)(offset * sizeof(T)));
	glEnableVertexAttribArray(idx);
	if (divisor > 0) {
		glVertexAttribDivisor(idx, divisor);
	}
}

void draw(VAO vao, GLenum mode, GLuint count, GLenum type, GLint indices, GLuint instanceCount = 1) {
	glBindVertexArray(vao.val);
	glDrawElementsInstanced(mode, count, type, (void*)indices, instanceCount);
}

void unbindBuffer(GLenum type) {
	glBindBuffer(type, 0);
}

void unbindVAO() {
	glBindVertexArray(0);
}

void cleanup(VAO vao) {
	glDeleteBuffers(1, &vao.posVBO);
	glDeleteBuffers(1, &vao.offsetVBO);
	glDeleteBuffers(1, &vao.sizeVBO);
	glDeleteBuffers(1, &vao.EBO);
	glDeleteVertexArrays(1, &vao.val);
}

void gen2DCircleArray(float*& vertices, unsigned int*& indices, unsigned int noTriangles, float radius = 0.5f) {
	vertices = new float[(noTriangles + 1) * 2];

	vertices[0] = 0.0f;
	vertices[1] = 0.0f;

	indices = new unsigned int[noTriangles * 3];

	float pi = 4 * atanf(1.0f);
	float noTrianglesF = (float)noTriangles;
	float theta = 0.0f;

	for (unsigned int i = 0; i < noTriangles; i++) {
		vertices[(i + i) * 2 + 0] = radius * cosf(theta);
		vertices[(i + i) * 2 + 1] = radius * sinf(theta);

		indices[i * 3 + 0] = 0;
		indices[i * 3 + 1] = i + 1;
		indices[i * 3 + 2] = i + 2;

		theta += 2 * pi / noTriangles;
	}

	indices[(noTriangles - 1) * 3 + 2] = 1;
}

void framebufferSizeCallback(GLFWwindow* window, int width, int height) {
	glViewport(0, 0, width, height);
	scrWidth = width;
	scrHeight = height;

	setOrthographicProjection(shaderProgram, 0, width, 0, height, 0.0f, 1.0f);

}


void processInput(GLFWwindow* window, double dt) {
	float* paddleOffsets;

	if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS) {
		glfwSetWindowShouldClose(window, true);
	}
	

	if (glfwGetKey(window, GLFW_KEY_W) == GLFW_PRESS) {
		if (paddleOffsets[0] < scrHeight - paddleBoundary) {
			paddleOffsets[0] += dt * paddleSpeed;
		}
		else {
			paddleOffsets[0] = scrHeight - paddleBoundary;
		}
	}
	if (glfwGetKey(window, GLFW_KEY_S) == GLFW_PRESS) {
		if (paddleOffsets[0] > paddleBoundary) {
			paddleOffsets[0] -= dt * paddleSpeed;
		}
		else {
			paddleOffsets[0] = scrHeight - paddleBoundary;
		}
	}

	if (glfwGetKey(window, GLFW_KEY_UP) == GLFW_PRESS) {
		if (paddleOffsets[1] < scrHeight - paddleBoundary) {
			paddleOffsets[1] += dt * paddleSpeed;
		}
		else {
			paddleOffsets[1] = scrHeight - paddleBoundary;

		}
	}

	if (glfwGetKey(window, GLFW_KEY_DOWN) == GLFW_PRESS) {
		if (paddleOffsets[1] > paddleBoundary) {
			paddleOffsets[1] -= dt * paddleSpeed;
		}
		else {
			paddleOffsets[1] = scrHeight - paddleBoundary;
		}
	}
}


void clearScreen() {
	glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
	glClear(GL_COLOR_BUFFER_BIT);
}

void newFrame(GLFWwindow* window) {
	glfwSwapBuffers(window);
	glfwPollEvents();
}

void cleanup() {
	glfwTerminate();
}

int main() {
	std::cout << "BEM VINDO AO PING PONG" << std::endl;

	double dt = 0.0;
	double lastFrame = 0.0;

	initGLFW(3, 3);

	GLFWwindow* window = nullptr;
	createWindow(window, title, scrWidth, scrHeight, framebufferSizeCallback);
	if (!window){
		std::cout << "Não foi possivel criar a janela" << std::endl;
		cleanup();
		return -1;
	}

	if (!loadGlad()) {
		std::cout << "Não foi possivel iniciar o GLAD" << std::endl;
		cleanup();
		return -1;
	}

	glViewport(0, 0, scrWidth, scrHeight);

	shaderProgram = genShaderProgram("main.vs", "main.fs");
	setOrthographicProjection(shaderProgram, 0, scrWidth, 0, scrHeight, 0.0f, 1.0f);


	float paddleVertices[] = {

		0.5f, 0.5f,
		-0.5, 0.5f,
		-0.5, -0.5f,
		0.5f, -0.5f
	};

	unsigned int paddleIndices[] = {
		0, 1, 2,
		2, 3, 0
	};

	paddleOffsets[0] = { 35.0f, scrHeight / 2.0f },
	paddleOffsets[1] = { scrWidth - 35.0f, scrHeight / 2.0f };

	vec2 paddleSizes[] = {
		paddleWidth, paddleHeight
	};


	VAO paddleVAO;
	genVAO(&paddleVAO);

	genBufferObject<float>(paddleVAO.posVBO, GL_ARRAY_BUFFER, 2 * 4, paddleVertices, GL_STATIC_DRAW);
	setAttPointer<float>(paddleVAO.posVBO, 0, 2, GL_FLOAT, 2, 0);

	genBufferObject<vec2>(paddleVAO.offsetVBO, GL_ARRAY_BUFFER, 2 * 2, paddleOffsets, GL_DYNAMIC_DRAW);
	setAttPointer<float>(paddleVAO.offsetVBO, 1, 2, GL_FLOAT, 2, 0, 1);

	genBufferObject<vec2>(paddleVAO.sizeVBO, GL_ARRAY_BUFFER, 2 * 2, paddleOffsets, GL_STATIC_DRAW);
	setAttPointer<float>(paddleVAO.sizeVBO, 2, 2, GL_FLOAT, 2, 0, 2);

	genBufferObject<GLuint>(paddleVAO.EBO, GL_ELEMENT_ARRAY_BUFFER, 2 * 4, paddleIndices, GL_STATIC_DRAW);

	unbindBuffer(GL_ARRAY_BUFFER);
	unbindVAO();


	float* ballVertices;
	unsigned int* ballIndices;
	unsigned int noTriangles = 50;
	gen2DCircleArray(ballVertices, ballIndices, noTriangles, 0.5f);

	ballOffset = { scrWidth / 2.0f, scrHeight / 2.0f	};

	vec2 ballSizes[] = {
		ballDiameter, ballDiameter
	};


	VAO ballVAO;
	genVAO(&ballVAO);

	genBufferObject<float>(ballVAO.posVBO, GL_ARRAY_BUFFER, 2 * (noTriangles + 1), ballVertices, GL_STATIC_DRAW);
	setAttPointer<float>(ballVAO.posVBO, 0, 2, GL_FLOAT, 2, 0);

	genBufferObject<vec2>(ballVAO.offsetVBO, GL_ARRAY_BUFFER, 1, &ballOffset, GL_DYNAMIC_DRAW);
	setAttPointer<float>(ballVAO.offsetVBO, 1, 2, GL_FLOAT, 2, 0, 1);

	genBufferObject<vec2>(ballVAO.sizeVBO, GL_ARRAY_BUFFER, 1 * 2, ballSizes, GL_STATIC_DRAW);
	setAttPointer<float>(ballVAO.sizeVBO, 2, 2, GL_FLOAT, 2, 0, 1);

	genBufferObject<unsigned int>(ballVAO.EBO, GL_ELEMENT_ARRAY_BUFFER, 3 * noTriangles, ballIndices, GL_STATIC_DRAW);

	unbindBuffer(GL_ARRAY_BUFFER);
	unbindVAO();


	while (!glfwWindowShouldClose(window)) {
		dt = glfwGetTime() - lastFrame;
		lastFrame += dt;

		processInput(window, dt);

		paddleOffsets[0].y += paddleVelocities[0] * dt;
		paddleOffsets[1].y += paddleVelocities[1] * dt;

		ballOffset.x += ballVelocity.x * dt;
		ballOffset.y += ballVelocity.y * dt;


		if (ballOffset.y - ballRadius <= 0 || ballOffset.y + ballRadius >= scrHeight) {
			ballVelocity.y *= -1;
		}

		bool reset = false;
		if (ballOffset.x - ballRadius <= 0) {
			std::cout << "Ponto pro jogador da direita" << std::endl;
			reset = true;
		}
		else if (ballOffset.x + ballRadius >= scrWidth) {
			std::cout << "Ponto pro jogador da esquerda" << std::endl;
			reset = true;
		}
		if (reset) {
			ballOffset.x = scrWidth / 2.0f;
			ballOffset.y = scrHeight / 2.0f;

			ballVelocity.x = initBallVelocity.x;
			ballVelocity.y = initBallVelocity.y;
		}

		int i = 0;
		if (ballOffset.x > scrHeight / 2.0f) {
			i++;
		}

		vec2 distance = { std::abs(ballOffset.x - paddleOffsets[i].x), std::abs(ballOffset.y - paddleOffsets[i].y) };

		if (distance.x <= halfPaddleWidth + ballRadius &&
			distance.y <= halfPaddleHeight + ballRadius) {
			bool collision = false;
			if (distance.x <= halfPaddleWidth) {
				collision = true;
				ballVelocity.y *= -1;
			}
			else if (distance.y <= halfPaddleHeight) {
				collision = true;
				ballVelocity.y *= -1;
			}

			if ((distance.x - halfPaddleWidth) * (distance.x - halfPaddleWidth) +
				(distance.y - halfPaddleHeight) * (distance.y - halfPaddleHeight)
				<= (ballRadius * ballRadius)) {

				collision = true;
				ballVelocity.x *= -1;
			}
			if (collision) {
				float k = 5.0f;
				ballVelocity.y += k * paddleVelocities[i];
			}
		}

		

		clearScreen();

		updateData<vec2>(paddleVAO.offsetVBO, 0, 2, paddleOffsets);
		updateData<vec2>(ballVAO.offsetVBO, 0, 1, &ballOffset);

		bindShader(shaderProgram);
		draw(paddleVAO, GL_TRIANGLES, 3 * 2, GL_UNSIGNED_INT, 0, 2);
		draw(ballVAO, GL_TRIANGLES, 3 * noTriangles, GL_UNSIGNED_INT, 0);

		newFrame(window);
	}

	cleanup(paddleVAO);
	cleanup(ballVAO);
	deleteShader(shaderProgram);
	cleanup();

	return 0;
}
