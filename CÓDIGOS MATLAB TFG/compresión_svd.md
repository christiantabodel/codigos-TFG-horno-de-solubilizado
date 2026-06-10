Este es el código 1


``` matlab
% COMPRESIÓN DE IMAGENES MEDIANTE SVD
% Comparativa: imagen original vs. aproximación de rango r=50

clear; close all; clc;

% Cargar la imagen
A = imread('horno2.jpg');
X = double(rgb2gray(A));
[nx, ny] = size(X);

% Calcular la SVD
[U, S, V] = svd(X);

% Truncamiento de rango
r = 50;
Xapprox = U(:,1:r) * S(1:r,1:r) * V(:,1:r)';

% Porcentaje de almacenamiento
porcentaje = r * (1 + nx + ny) / (nx * ny) * 100;
fprintf('r = %d: almacenamiento = %.2f%%\n', r, porcentaje);

% Energía acumulada
energia = cumsum(diag(S)) ./ sum(diag(S));
fprintf('Energia acumulada para r = %d: %.2f%%\n', r, energia(r)*100);

% Figura 1: Comparativa original vs. truncada
figure;
subplot(1,2,1);
imagesc(X); axis off; axis image; colormap gray;
title('Original','FontSize',12,'FontWeight','bold');

subplot(1,2,2);
imagesc(Xapprox); axis off; axis image; colormap gray;
title(sprintf('r=%d, %.2f%% almacenamiento', r, porcentaje), ...
      'FontSize',12,'FontWeight','bold');

set(gcf, 'Color', 'w', 'Position', [100 100 800 350]);
ax1 = subplot(1,2,1); set(ax1, 'Position', [0.05 0.08 0.42 0.82]);
ax2 = subplot(1,2,2); set(ax2, 'Position', [0.53 0.08 0.42 0.82]);

% Figura 2: Valores singulares y energia acumulada
figure;

subplot(1,2,1);
semilogy(diag(S), 'b', 'LineWidth', 1.5);
xlabel('Indice r','FontSize',12);
ylabel('\sigma_r','FontSize',14);
title('Valores Singulares','FontSize',12);
grid on;
xlim([-20 min(nx,ny)]);

subplot(1,2,2);
plot(cumsum(diag(S)) ./ sum(diag(S)), 'b', 'LineWidth', 1.5);
xlabel('Indice r','FontSize',12);
ylabel('Suma acumulada / Total','FontSize',14);
title('Energia acumulada','FontSize',12);
grid on;
ylim([0 1]);
xlim([-20 min(nx,ny)]);

set(gcf, 'Color', 'w', 'Position', [100 100 900 350]);
```
