FROM mcr.microsoft.com/dotnet/core/sdk:3.0 AS build
WORKDIR /code
COPY . .
RUN dotnet restore
RUN dotnet publish --output /output --configuration Release

FROM mcr.microsoft.com/dotnet/core/aspnet:3.0
COPY --from=build /output /app
WORKDIR /app
ENTRYPOINT ["dotnet", "k8sdemo.dll"]